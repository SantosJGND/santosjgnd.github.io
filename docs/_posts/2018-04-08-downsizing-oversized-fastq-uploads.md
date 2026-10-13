---
layout: post
title: "Downsizing oversized FASTQ uploads: a ratio, a fudge factor, and a failure check"
date: 2018-04-08 19:11:55 +0100
categories: python django fastq
---

Users upload FASTQ files that are much bigger than the pipeline can comfortably process. Rejecting them is annoying
for the user, and running them whole is slow and expensive for us. So the portal downsamples large files at random
before analysis. I wanted to know exactly how that decision is made, so this post walks through the settings, the
downsizing function and the place where it is called.

### The limits

There are two settings, both read from the environment with defaults:

```python
MAX_FASTQ_FILE_UPLOAD = config("MAX_FASTQ_FILE_UPLOAD", default=50971520, cast=int)   ### 50M
DOWN_SIZE_FASTQ_FILES = config("DOWN_SIZE_FASTQ_FILES", default=True, cast=bool)
MAX_FASTQ_FILE_WITH_DOWNSIZE = config("MAX_FASTQ_FILE_WITH_DOWNSIZE", default=250971520, cast=int)  ### 250M
```

The comments explain the intent. With downsizing on, we accept files up to about 250 MB and reduce them to the 50 MB
target. With it off, the cap is the smaller limit. The web form tells users which range they are in.

### Deciding whether to downsize

The function `make_downsize` measures the larger of the two paired files and compares it to the target:

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

The ratio is the fraction of reads to keep. I did not like the small correction when I first read it. Adding 0.1
to any ratio below 0.8 means the output comes out larger than the target. The code gives no reason for it. As far as I
can tell it is a patch to the output size, not a model of how file size relates to read count.

### Running the sampler

The reads are sampled with an external tool, and the ratio is passed in with two decimal places:

```python
cmd = "{} -p {:.2f} -o {}/sample {}".format(
    self.software_names.get_fastqtools_sample(), ratio, path_to_work, file_names,
)
exist_status = os.system(cmd)
if exist_status != 0:
    self.logger_production.error("Fail to run: " + cmd)
    raise Exception("Fail to downsize")
```

Gzipped inputs are decompressed first and compressed again at the end. Failure is detected from the return code of
`os.system`, and that is the only check. The sampled files are then used in place of the originals.

### Where it is called

Downsizing happens before the quality trimming step. If the files were reduced, the outputs replace the originals,
and the sample is marked with an alert. The web page then shows a "Sample was downsized" note based on that flag. The
nanopore path does the same thing with its own copy of the logic.

### What I would question now

Size is measured on disk, not in reads. A file with long headers or long quality strings is penalised more than one
with short ones, so the ratio is not a read budget. Measuring the output after sampling and adjusting once would be
more honest than a fixed offset. The two limits are also close enough that changing one without the other can
quietly switch downsizing off for a whole class of uploads.

A failed sample stops the whole job rather than skipping that one file, which is right for the pipeline but not for a
large batch where one bad file should not block the rest. The repository also has a test for `make_downsize`, which
is the only thing I could find that checks this behaviour.

The date on this post is the commit that added the function. The upload code around it has changed since, so trust
the current code over this description.
