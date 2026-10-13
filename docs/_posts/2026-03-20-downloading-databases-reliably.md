---
layout: post
title: "Downloading multi-GB databases: what is verified, what is retried, and what is not"
date: 2026-03-20 17:06:41 +0000
categories: python downloads integrity
---

Some of TELEVIR's reference databases are several gigabytes. A download that fails halfway, or a file that
arrives corrupted, can leave an install that looks complete and is not. This post checks what the install code
actually does for integrity, retries, and locking, and lists the gaps. The code is in
`install_scripts/modules/db_install.py` and `install_scripts/bin/centrifuge-download`.

### Integrity by extension

The dispatcher `verify_file_integrity` picks a test from the file extension:

- `.gz`, `.tar.gz`, `.tgz`: `gzip -t`
- `.xz`: `xz -t`
- anything else: returns `True` with no check

A failed check deletes the file and reports it as corrupted. The gzip check is a real stream test:

```python
subprocess.run(["gzip", "-t", filepath], capture_output=True, check=True)
```

Two gaps follow from the extension rule. A plain `.tar` or `.fa` is never checked. And there are no checksums
anywhere in the install scripts: no md5 and no sha256, so a file that is valid gzip but the wrong version
passes.

### The generic download

The filter downloader (`dl_filter_file`) substitutes `{version}` in the URL and filename, then calls wget:

```python
subprocess.run(["wget", url, "-O", filepath], check=False)
```

Two things stand out. `check=False` means the exit code of wget is ignored, so verification is the only gate.
And `-O` writes straight to the final path, with no temporary file. A partial download sits at the real name
until the next run's `gzip -t` removes it. There are no retries.

### The Centrifuge downloader

`centrifuge-download` is a bash script, and it does retry. Each file is fetched with the same command written
three times in a row:

```bash
$DL_CMD "$LIBDIR/$DOMAIN/$NAME.gz" "$FILEPATH" || \
$DL_CMD "$LIBDIR/$DOMAIN/$NAME.gz" "$FILEPATH" || \
$DL_CMD "$LIBDIR/$DOMAIN/$NAME.gz" "$FILEPATH" || \
{ printf "\nError downloading $FILEPATH!\n" >&2 && exit 1; }
```

There is no sleep between attempts, so the three tries happen back to back. The only integrity check is the
`gunzip -f`, which relies on gzip's own CRC at decompression. Files already present and non-empty are skipped,
which is the only resume behaviour.

Failures in one file are swallowed by `download_n_process_nofail`, which runs the download with `|| true`.

### Locking

This is the only `flock` in the repository. It guards the header-to-taxid lines that parallel workers write to
stdout:

```bash
... | flock -x $LOCK_FILE -c 'cat'
```

The lock is only set when `N_PROC > 1`, and `flock` is required for parallel runs. Without `flock`, the script
forces one process. The lock file is `./lockfile.lock`, relative to the directory the script was invoked from,
and it is removed at the end of a normal run. `db_install.py` does not lock anything itself, so two installs
running at once into the same directory are not protected.

### Other things to fix

- **TLS is turned off for one fallback.** When the taxdump is not set, `centrifuge_install` falls back to
  `wget --no-check-certificate`. That disables certificate checks for that download.
- **Some `os.system` calls ignore failure.** The prebuilt Centrifuge path runs `wget`, `tar`, and `rm` through
  `os.system`, so a failed download is not raised.
- **The prebuilt path is hard-coded.** The registration uses `/opt/data/prebuilt/` and ignores the path from the
  YAML.

### What I would do

Verify every download type, including plain tar, and store a checksum per entry in the YAML. Download to a
temporary name and rename on success. Use `check=True` and a real retry with backoff in one shared helper, so
the Python and bash paths behave the same. Take a lock per output directory in `db_install.py`, not only in the
Centrifuge script.

The YAML side of the same installs is in [the declarative installs post]({% post_url 2026-04-07-declarative-bioinformatics-installs %}).
