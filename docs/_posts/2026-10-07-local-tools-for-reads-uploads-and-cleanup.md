---
layout: post
title: "A bioinformatician's toolbelt: small scripts for reads, uploads and cleanup"
date: 2026-10-07 13:23:13 +0100
categories: bash python bioinformatics tooling
---

Every bioinformatician ends up with a folder of small scripts. Nothing in it is clever. One script strips a
suffix from read names, another pulls the run IDs out of a directory, another gets a sample set down to a
manageable size. They are the unglamorous part of the work, and they eat more hours than the analysis does.
Mine lives in a folder I put on my `PATH`, and it has grown one annoyance at a time. This post goes through the
toolbelt by job, and records what I tested and what I only read.

The folder is not a git repository, so the date on this post is the modification time of `purge_reads`, the
newest script in it. I have left out two helpers: one reads a VPN credential file, and the other only makes
sense on one machine.

### Tidying reads

Some read names end in `/1` or `/2`, which downstream tools do not expect. `clean_read_name` removes those
suffixes. It only edits header lines, which are every fourth line of a FASTQ file, so a quality line that
happens to start with `@` is never touched:

```bash
awk 'NR % 4 == 1 { sub(/\/[12]$/, "") } { print }' "$FASTQ" > "$TMP"
cat "$TMP" > "$FASTQ"
```

Writing back through `cat` keeps the file's permissions and owner. It only works on plain FASTQ, which is
enough for what I use it for.

For Nanopore runs, `fastq_runids_map` prints each file next to the run ID in its first header. An empty
directory used to make it pass the literal pattern `*.fastq.gz` to `zgrep`. The README's change log records
the `nullglob` fix for that. A missing run ID comes out as an empty field rather than an error, so I look at
the output before I trust it.

### Making data smaller

Sometimes a sample is simply too big for the job. `sample_to_size` takes a target size in bytes and computes
a fraction of the input to keep. An earlier version used `bc` with two decimal places. That rounded small
fractions to zero and gave an empty file. Six decimals fixes it: 0.0042 is 0.00 at two places and 0.004200 at
six.

There is a trap in `seqtk`, which treats any value of 1 or more as a number of reads, not a fraction. The
script copies the file when the target is at least the input size, so that never reaches `seqtk`. Sampling
uses a fixed seed, so the same input gives the same reads. I took the `seqtk` rule from its documentation and
did not reproduce it myself.

### Getting data in

Two scripts fetch public reads. `dl_accession_reads` takes one SRA run. It drops reads under 100 bp, skips
technical reads, splits pairs into two files, and writes uncompressed into the current directory. It checks
that the toolkit binary exists before it starts, which saves a confusing error later.

`dl_bioproject` takes a whole BioProject and downloads every run in it:

```bash
esearch -db sra -query $BIOPROJECT | efetch -format runinfo | cut -d ',' -f 1 | grep SRR | \
xargs $SRATOOLKIT_BIN/bin/fastq-dump --split-files --gzip -O $OUTPUT_DIR
```

It fails more quietly than it should. The script defines an `ENTREZ_BINDIR` variable and never uses it, so it
runs whichever `esearch` comes first on the path. I confirmed that with `grep`. There is no `pipefail`, so the
exit status is that of the last command, and a failed search can go unnoticed. (I read this and did not run it
against the network.) The `grep SRR` step also drops `ERR` and `DRR` runs, the European and Japanese prefixes,
without a message. (Also from reading.) The header mentions `-N` and `-X` flags for downloading part of a run.
They are not parsed, so they do nothing, and the README says so.

### Preparing an upload

Before a sample set goes to INSaFLU or Televir, each sample needs one row: a name, a forward file and a reverse
file. Two Python scripts do this, `create_televir_upload.py` for the table and `rename_for_upload.py` for copying
files to ordered names. Both use the same pair detection, a character trie over file names that pairs two files at
their lowest common ancestor. I tried three naming styles, `_R1`/`_R2`, `_1`/`_2`, and Illumina
`s1_S1_L001_R1_001.fastq.gz`, and all three paired correctly.

They handle leftovers differently, and that surprised me. I gave both scripts two pairs and one file with no mate.
The table script wrote two rows, exited normally, and logged `Found 2 fastq files` for five files. The count was
pairs, and the orphan disappeared without a word. The renaming script stopped and named the orphan. The table
script never runs the check the renaming script runs, so an unmatched read can reach an upload without its mate.

The paired-end default is another trap. The table script's `--pe` defaults to `none`, and anything other than
`"no"` takes the paired path. Three single-end files with the default setting produced `No fastq files found`,
even though the files were there. The launcher pins `-pe=no`, so this only happens when you call the Python script
directly. The README calls that intentional, but the message does not explain it.

### Removing human reads

Before a sample goes into a pathogen pipeline, I remove the reads that map to the human genome. Two scripts do
it, one for paired-end data and one for single-end. Both map to GRCh38 with `bwa mem`, keep the unmapped reads,
and convert them back to FASTQ.

The single-end script used to keep the mapped reads, which are the ones it was supposed to throw away. The paired
script already had the right flag. The current code uses `-f 4` in both, and the README's change log records the
fix:

```bash
$SAMTOOLS_BIN view -b -f 4 "${FASTQ_PREFIX}.bam" > "${FASTQ_PREFIX}_depleted.bam"
```

What bothers me about that bug is not `samtools`. Two scripts doing the same job had opposite flags, and nothing
compared them. One test with a mapped read and an unmapped read would have caught it.

The scripts also skip any step whose output already exists. That makes them resumable, until a run is killed
halfway. The output file appears as soon as the step starts, so the next run finds a truncated BAM and carries on
with it, printing only "Using existing". I read this in the code and did not reproduce it with a real reference. The
safer habit is to write to a temporary name and rename on success:

```bash
bwa mem -t 4 "$REF" "$FASTQ" | samtools view -b -o "$tmp" && mv "$tmp" "$out"
```

The paired script does one thing right: orphan reads, whose mate was removed, go to a separate singletons file, so
R1 and R2 stay the same length.

### Cleaning up

`purge_reads` deletes read files, so it is the tool I was most careful with. It refuses to run on `/` or on a
missing directory. The default is a dry run that lists the matches with subtotals. Deleting needs `-delete` and a
`y/N` prompt. Without a terminal it stops unless `-yes` is given, so a stray pipe cannot delete anything. Symlinks
are never followed. The README reports a 43-case test suite that I did not run again.

Before each deletion, it writes a `.txt` placeholder beside the file, with the original path, size, times, and a
note that the file was deleted. Repeat records are appended. The README says that if the record cannot be written,
the file is kept, and that holds.

The reverse case does not. I tested a file in a read-only directory with a placeholder already in place. The
removal failed, the run log recorded `FAIL-RM`, the exit code was 2, and the file was still there. The placeholder,
though, now said the file had been deleted, with a timestamp. Anyone reading that record later would be misled, and
only the run log shows the failure. The fix is to write the record as pending, remove the file, and mark it deleted
only if the removal worked.

### What a toolbelt needs

Looking at the whole folder, the same few rules keep coming up. A tool that writes a table or a file list for another
system should be strict about what it leaves out, because nobody reads the rows that never appear. Two tools doing the
same job should share one definition of it, and a test should cover the flag or rule they both depend on. Anything that
writes to disk should write to a temporary name and rename on success. And an audit record should say what happened, not
what was planned. None of this is new, but a folder of helpers is where these habits either stick or quietly rot.
