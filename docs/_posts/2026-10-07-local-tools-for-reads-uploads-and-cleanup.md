---
layout: post
title: "A folder of read tools: preparing, depleting, downloading and purging reads"
date: 2026-10-07 13:23:13 +0100
categories: bash python bioinformatics tooling
---

Most of the day-to-day read handling I do is not done by the pipeline. It is done by a flat folder of small scripts that
I keep for myself. They clean read names, subsample files that are too big, prepare upload tables, remove human reads,
download public data and delete old files. Each script is short, but several have details that are easy to get wrong, and
I only found some of them by testing. This post goes through the folder section by section, in the same order as its README.

The folder is not a git repository, so the date on this post is the modification time of `purge_reads`, the newest script
in the set. I have left out the infrastructure helpers: one reads a VPN credential file, and the other is specific to one
machine.

## Read preparation

Three small helpers run before reads go anywhere else. The first strips mate markers. Some read names end in `/1` or `/2`,
which the downstream tools do not expect, so `clean_read_name` removes them. It only touches header lines, which are every
fourth line in a FASTQ file:

```bash
awk 'NR % 4 == 1 { sub(/\/[12]$/, "") } { print }' "$FASTQ" > "$TMP"
cat "$TMP" > "$FASTQ"
```

A quality line that happens to start with `@` is never changed. Writing back through `cat` keeps the file's permissions and
owner. It only works on plain FASTQ, which is fine for what I use it for.

The second helper, `fastq_runids_map`, is for Nanopore data. It prints each file with the run ID from its first header. I
added `nullglob` after finding that an empty directory made it pass the literal pattern `*.fastq.gz` to `zgrep`. A missing
run ID comes out as an empty field rather than an error, so I check the output before trusting it.

The third is subsampling. `sample_to_size` takes a target size in bytes and works out a fraction of the input. My first
version used `bc` with two decimal places, which rounded small fractions to zero and gave an empty file. Six decimals fixes
it: 0.0042 is 0.00 at two places and 0.004200 at six. There is also a trap in `seqtk`, which treats any value of 1 or more
as a number of reads rather than a fraction. The script copies the file if the target is at least the input size, so that
never happens. Sampling uses a fixed seed, so repeat runs give the same reads. I took the `seqtk` behaviour from its
documentation and did not reproduce it myself.

## Upload preparation

Before a sample set goes to INSaFLU or Televir, each sample needs a row: a name, a forward file and a reverse file. There
are two Python scripts for this, and each has a shell launcher. `create_televir_upload.py` writes the tab-separated table.
`rename_for_upload.py` copies the files to ordered names and writes a mapping file. Both use the same pair detection, which
builds a character trie over the file names and pairs two files at their lowest common ancestor. I tried three naming styles,
`_R1`/`_R2`, `_1`/`_2` and Illumina names like `s1_S1_L001_R1_001.fastq.gz`, and all three paired correctly.

The two scripts handle leftovers differently, and that was the part that surprised me. I gave both of them two pairs and one
file with no mate. The table script wrote two rows, exited normally, and logged `Found 2 fastq files` for a directory with five
files in it. The count is pairs, not files, and the orphan disappeared without a word. The renaming script stopped with an
error that named the orphan. The table script never runs the check that the renaming script does, so a sample set with an
unmatched read ends up in the upload with a missing mate.

The paired-end default is another trap. The table script defaults `--pe` to `none`, and anything other than `"no"` takes the
paired path. I gave it three single-end files with the default setting, and it reported `No fastq files found`, even though
the files were right there. The paired search ran, found no pairs, and reported that as an empty directory. The launcher
pins `-pe=no`, so this only happens if you call the Python script directly. The README calls the difference intentional, but
the message does not say what it is.

## Host depletion

Two scripts remove human reads from a sample before it goes into a pathogen pipeline: one for paired-end data and one for
single-end. Both map reads to GRCh38 with `bwa mem`, keep the reads that did not map, and convert them back to FASTQ.

The single-end script used to have a flag that was the wrong way round. It kept mapped reads, which are exactly the human
reads it was supposed to remove. The paired script already had the right flag, and the README's change log records the fix.
The current code uses `-f 4` in both, which keeps the unmapped reads:

```bash
$SAMTOOLS_BIN view -b -f 4 "${FASTQ_PREFIX}.bam" > "${FASTQ_PREFIX}_depleted.bam"
```

What bothers me about that bug is not `samtools`. It is that two scripts doing the same job had opposite flags, and nothing
compared them. One test with a mapped read and an unmapped read would have shown it.

The scripts also skip any step whose output file already exists. The README calls this resumable, which is true in the good
case. The problem is that the output file appears as soon as a step starts. If a run is killed halfway, the next run finds a
truncated BAM and carries on with it, printing only "Using existing". I read this in the code and did not reproduce it with a
real reference. The safer pattern is to write to a temporary name and rename it only when the step succeeds:

```bash
bwa mem -t 4 "$REF" "$FASTQ" | samtools view -b -o "$tmp" && mv "$tmp" "$out"
```

Some smaller things I noticed: the reference path is fixed under a single user's home directory, the thread count for `bwa`
is fixed at four, and the reference index is built on first use with no lock. Two runs started at the same moment could both try
to build it. The paired script also does something useful: orphan reads, whose mate was removed, go into a separate singletons
file, so the R1 and R2 files stay the same length.

## Downloads

Two scripts fetch public reads. `dl_accession_reads` fetches one run with the SRA Toolkit. It drops reads shorter than 100 bp,
skips technical reads, and splits paired reads into two files. The output is uncompressed and goes into the current directory.
It checks that the toolkit binary exists before it starts, which I am glad of.

`dl_bioproject` takes a whole BioProject and downloads every run in it. It is a pipeline of five commands:

```bash
esearch -db sra -query $BIOPROJECT | efetch -format runinfo | cut -d ',' -f 1 | grep SRR | \
xargs $SRATOOLKIT_BIN/bin/fastq-dump --split-files --gzip -O $OUTPUT_DIR
```

I think it fails more quietly than it should, and I can point to three reasons. The script defines an `ENTREZ_BINDIR` variable
and never uses it, so it calls whichever `esearch` is first on the path. There is no `pipefail`, so the exit status is that of
the last command, which means a failed search can go unnoticed. (I read this in the code and did not run it against the
network.) And the `grep SRR` step drops any run that starts with `ERR` or `DRR`, which are the European and Japanese prefixes,
without saying so. (Also from reading the code.)

The header mentions `-N` and `-X` flags for downloading only part of a run. The script does not parse them, so they do nothing,
and the README says the same. The two scripts also write their output differently: one uncompressed into the current directory,
the other gzipped into a named one. Both are in use, so I have not changed either.

## Cleanup

`purge_reads` removes read files under a directory, and it is the script I was most careful with. It has several guards that I
checked. It refuses to run on `/` and on a directory that does not exist. The default is a dry run, which lists the matches with
subtotals per directory and changes nothing. Deleting needs the `-delete` flag and then a `y/N` prompt, and without a terminal on
standard input it stops unless `-yes` is given, so a stray pipe cannot delete anything. Symlinks are never followed and directories
are never touched. The README reports a 43-case test suite run in a temporary directory. I did not run it again.

Before a file is removed, the script writes a `.txt` placeholder beside it, with the original path, the size, the modification
time and the deletion time. If the file has been purged before, the new record is appended. The order is: write the record,
then remove the file. The README says that if the record cannot be written, the file is not deleted, and that holds.

The reverse does not. I tested a file in a directory I had made read-only, with a placeholder already in place. The delete failed
with a permission error, the run log recorded `FAIL-RM`, the exit code was 2, and the file was still there. But the placeholder
now said that the file had been deleted, with a timestamp and everything. Anyone reading the `.txt` file later would be misled,
and only the run log shows the failure. I would change the order: write the record as pending, remove the file, and then mark
the record as deleted only if the removal worked.

## What I would change, across the folder

A tool that writes a table or a file list for another system should be strict about what it leaves out. The upload table script
should run the same leftover check as the renaming script, and it should count files rather than pairs. The depletion scripts
should write to temporary names and rename on success, keep the flag choice in one place with a test, and take the reference path
from configuration. The download scripts should fail when a search returns nothing, use the Entrez variable or delete it, and
keep all the accession prefixes they claim to support. And the audit record should describe what actually happened, not what was
planned. Writing the plan first is fine, as long as the record is updated after the outcome is known.
