---
layout: post
title: "Downloading multi-GB databases: what is verified, what is retried, and what is not"
date: 2026-03-20 17:06:41 +0000
categories: python downloads integrity
---

Some of the reference databases TELEVIR uses are several gigabytes. A download that fails halfway, or a file that
arrives corrupted, can leave an install that looks complete and is not. I wanted to know what our install code really
does about that, so I read the integrity checks, the retries and the locking, and then wrote down the gaps I found.

### Checking the file by its extension

The integrity check picks a test from the file extension. Gzip files, including `.tar.gz`, are checked with `gzip -t`,
and xz files with `xz -t`. Anything else is accepted without a check. A failed check deletes the file and reports it
as corrupted. The gzip test is a real check of the whole stream, not just the header:

```python
subprocess.run(["gzip", "-t", filepath], capture_output=True, check=True)
```

That rule has two holes. A plain `.tar` or `.fa` file is never checked. And there are no checksums anywhere in the
install scripts: no md5, no sha256. So a file that is valid gzip but the wrong release passes.

### The generic download

The generic downloader substitutes the version into the URL and filename, and then calls `wget`:

```python
subprocess.run(["wget", url, "-O", filepath], check=False)
```

Two things stood out to me. The `check=False` means the exit code of `wget` is ignored, so the integrity check afterwards
is the only thing standing between a failed download and a broken install. And `-O` writes straight to the final path,
with no temporary file. A partial download sits under its real name until the next run's `gzip -t` removes it. There
are no retries at all.

### The Centrifuge script does retry

The Centrifuge downloader is a bash script, and it does retry. Each file is requested with the same command written
three times in a row, so the attempts happen back to back:

```bash
$DL_CMD "$LIBDIR/$DOMAIN/$NAME.gz" "$FILEPATH" || \
$DL_CMD "$LIBDIR/$DOMAIN/$NAME.gz" "$FILEPATH" || \
$DL_CMD "$LIBDIR/$DOMAIN/$NAME.gz" "$FILEPATH" || \
{ printf "\nError downloading $FILEPATH!\n" >&2 && exit 1; }
```

There is no pause between attempts, so a flaky server gets three quick tries and then the job fails. The only integrity
check is the `gunzip` at the end, which relies on gzip's own checksum at decompression. Files that already exist and are
not empty are skipped, which is the only way it resumes. Failures for one file are swallowed by a wrapper that runs the
download with `|| true`, so one bad file does not stop the others, and it does not tell you about the bad file either.

### Locking

This is the only `flock` in the whole repository. It protects the header-to-taxid lines that parallel workers write to
standard output, so that their lines do not interleave:

```bash
... | flock -x $LOCK_FILE -c 'cat'
```

The lock is only set up when more than one process runs. Without `flock`, the script forces a single process. The lock
file is relative to the directory the script was started from, which I found surprising, and it is removed at the end of
a normal run. The Python install code does not lock anything itself, so two installs writing to the same directory at the
same time are not protected.

### Other things I would fix

When the taxonomy dump is missing, the Centrifuge build falls back to `wget` with certificate checking turned off. That is
a convenience that quietly removes a security check, and I would rather it failed. The prebuilt Centrifuge path runs its
download, extract and cleanup steps through `os.system`, so a failure there is never raised. And the prebuilt location is
fixed in code, ignoring the path in the configuration file.

### What I would do

Check every download type, including plain tar, and store a checksum for each entry in the configuration file. Download to
a temporary name and rename only on success. Use one shared helper for retries with a pause between attempts, so the Python
and bash paths behave the same. And take a lock per output directory in the Python code, not only in the Centrifuge script.

The configuration side of these installs is in
[the declarative installs post]({% post_url 2026-04-07-declarative-bioinformatics-installs %}).
