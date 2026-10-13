---
layout: post
title: "Declarative bioinformatics installs: one YAML file, and where it stops"
date: 2026-04-07 15:36:53 +0100
categories: python packaging bioinformatics
---

TELEVIR needs dozens of reference databases, each with its own URL, filename, and version. Hard-coding those
in Python means a code change for every new database. The project moved the description of each database into
one file, `sources.yaml`, so that most additions need no code. This post shows the format, how the loader
reads it, and where the "declarative" claim stops being true.

### The format

Each database is a named entry under a category. Here is a versioned entry, with the version in the URL:

```yaml
rvdb:
  url: "https://rvdb-prot.pasteur.fr/files/U-RVDBv{version}-prot.fasta.xz"
  description: "Reverse Virome Database"
  file: "U-RVDBv{version}-prot.fasta.xz"
  version_default: "22.0"
  install: false
  install_types: ["prot"]
  db_key: "prot"
```

A RefSeq entry names a download method instead of a single URL:

```yaml
viral_genome:
  url: "https://ftp.ncbi.nlm.nih.gov/refseq/release/viral/"
  file_pattern: "viral.genome.fna.gz"
  install: true
  install_types: ["nuc"]
  db_key: "nuc"
  download_method: "refseq"
  refseq_organism: "viral"
  refseq_type: "genome"
```

The keys the loader uses are `url`, `file`, `version` or `version_default`, `install`, `install_types`, and
`db_key`. `install: false` disables an entry, and a missing `install` key counts as enabled.

### The loader

`SourceLoader` (`install_scripts/load_sources.py`) is a singleton. It finds `sources.yaml` at the repo root and
parses it once. Accessors then answer questions about each entry: whether it is enabled, its file name, its
version, its database key, and its download method. Version lookup falls back in order: the `version` field,
then a version pulled out of the filename, then out of the URL.

The download plan is built by `list_all_db_configs`, which walks the categories and attaches the keys. The
main install routine (`main_install.py`, `prep_dl`) then decides how each entry is fetched:

```python
download_categories = ["ribosomal_rna", "protein", "nucleotide", "refseq"]
```

A RefSeq category goes to a RefSeq downloader. Every other category goes to a generic filter downloader that
substitutes `{version}` in the URL and filename, then downloads and verifies the file.

### Adding a database without code

- A new entry under `ribosomal_rna`, `protein`, or `nucleotide` is picked up automatically, provided it has a
  `url`, a `file`, and a `db_key` or `install_types`.
- A new RefSeq entry needs its `download_method`, `refseq_organism`, and `refseq_type`.

That is the part that is declarative.

### Where it is not

- **A new category needs code.** `prep_dl` only downloads the four categories in its list. An entry under any
  other name is silently never fetched.
- **Software needs code.** Tools in `software` (archives, git repos, conda tools) are mapped to environment
  keys by a dictionary in `install_source.py` (`software_mapping`). A new tool is a code edit.
- **Kraken2, Centrifuge, and Kaiju have their own functions.** Their entries are iterated by name, but each one
  calls hard-coded per-tool code. I did not verify that a new Kaiju entry works end to end.
- **Some keys are never read.** `version_template` appears in the YAML, and no Python reads it.
- **One path ignores the YAML.** The Centrifuge prebuilt registration hard-codes `/opt/data/prebuilt/`, and the
  YAML-driven path lookup is commented out.

### What I would do

Move the category list and the software mapping into the YAML too, so that the loader decides what to fetch
from the file alone. Delete the keys nothing reads. Add a test that loads every entry and checks that its
category is handled, so a silent skip becomes a failing test.

The download side of this is covered in [the database download post]({% post_url 2026-03-20-downloading-databases-reliably %}).
