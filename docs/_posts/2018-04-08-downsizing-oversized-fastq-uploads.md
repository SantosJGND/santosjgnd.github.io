---
layout: post
title: "Downsizing oversized FASTQ uploads: a ratio, a fudge factor, and a failure check"
date: 2018-04-08 19:11:55 +0100
categories: python django fastq
---

Users upload FASTQ files that are much larger than the pipeline can comfortably process. Rejecting them is
annoying, and running them whole is slow. INSaFLU instead randomly downsamples files above a size limit before
analysis. This post covers how that is decided and run, from `utils/software.py` and `utils/settings.py`.

### The settings

Two limits are defined in `utils/settings.py:81-92`:

```python
MAX_FASTQ_FILE_UPLOAD = config("MAX_FASTQ_FILE_UPLOAD", default=50971520, cast=int)   ### 50M
DOWN_SIZE_FASTQ_FILES = config("DOWN_SIZE_FASTQ_FILES", default=True, cast=bool)
MAX_FASTQ_FILE_WITH_DOWNSIZE = config("MAX_FASTQ_FILE_WITH_DOWNSIZE", default=250971520, cast=int)  ### 250M
```

The comments explain the intent. Files up to 250 MB are accepted when downsizing is on, and they are reduced
to the 50 MB target. Without downsizing, the cap is the smaller one.

### The decision

`make_downsize` (`utils/software.py:4862`) measures the larger of the two paired files and compares it to the
target:

```python
file_size_max = os.path.getsize(path_1)
if path_2 is not None and os.path.exists(path_2) and file_size_max < os.path.getsize(path_2):
    file_size_max = os.path.getsize(path_2)

if file_size_max > max_fastq_file:
    ratio = max_fastq_file / file_size_max
    ## small correction
    if ratio < 0.8:
        ratio += 0.1
```

The ratio is the fraction of reads to keep. The "small correction" adds 0.1 when the ratio is below 0.8.

### The command

The reads are sampled with an external tool:

```python
cmd = "{} -p {:.2f} -o {}/sample {}".format(
    self.software_names.get_fastqtools_sample(), ratio, path_to_work, file_names,
)
exist_status = os.system(cmd)
if exist_status != 0:
    self.logger_production.error("Fail to run: " + cmd)
    raise Exception("Fail to downsize")
```

Gzipped inputs are decompressed first and the outputs are compressed again. Failure is detected from the
return code of `os.system`, which is the only check.

### The call site

Downsizing runs before quality trimming (`run_fastq_and_trimmomatic`, `utils/software.py:3526-3532`). If the
files were reduced, the outputs replace the originals, and a sample-level alert is recorded. The web interface
shows a "Sample was downsized" tooltip based on that alert. The nanopore path (`software_minion.py:247-274`)
does the same thing.

### Things I would question today

- **The +0.1 correction is ad hoc.** Adding 0.1 to any ratio below 0.8 means the output is larger than the
  target. The ratio is not a size target, since the size of a sampled gzip is not linear in read count.
  Measuring the output and adjusting once would be more honest than a fixed offset.
- **Size is measured on disk, not in reads.** A file with long headers or quality lines is penalised.
- **The two limits are close to each other.** Changing one without the other can make downsizing silently
  stop applying.
- **Failure is an exception, not a skipped sample.** That is correct for the pipeline but not for a batch of
  uploads where one bad file would stop the rest.
- The repo has no `make_filter` or `downsample` function. My earlier notes assumed one existed. The only
  downsizing code is `make_downsize`, and it is covered by a test (`tests_insaflu/tests_software.py`).

The date on this post is the commit that added the downsizing function. The upload flow around it has changed
since, so read the code, not the post, for current behaviour.
