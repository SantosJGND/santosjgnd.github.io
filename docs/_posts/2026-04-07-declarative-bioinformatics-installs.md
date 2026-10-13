---
layout: post
title: "Declarative bioinformatics installs: one YAML file, and where it stops"
date: 2026-04-07 15:36:53 +0100
categories: python packaging bioinformatics
---

TELEVIR needs dozens of reference databases, and each one has its own URL, filename and version. For a long time, the
download code had those details hard-coded, which meant every new database was a code change. I wanted to move the
description of each database into one file, so that most additions need no code at all. This post shows the format, how
the loader reads it, and where the "no code needed" claim stops being true.

### An entry

Each database is a named entry under a category. Here is one with a version in the URL, which is the common case:

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

RefSeq is different, because the files are chosen by a pattern rather than a single URL, so the entry names a
download method instead:

```yaml
viral_genome:
  url: "https://ftp.ncbi.nlm.nih.gov/refseq/release/viral/"
  file_pattern: "viral.genome.fna.gz"
  install: true
  download_method: "refseq"
  refseq_organism: "viral"
  refseq_type: "genome"
```

Setting `install: false` turns an entry off. If the key is missing, the entry is treated as enabled, which I did not
expect when I first read it.

### The loader

`SourceLoader` reads the file once and answers questions about each entry: whether it is enabled, its filename, its
version, its database key, and how to download it. The version lookup has a fallback chain. It uses the `version`
field if there is one, then tries to pull a version out of the filename, then out of the URL.

The download plan is built by walking the categories and collecting the entries. The install routine then decides how
each category is fetched:

```python
download_categories = ["ribosomal_rna", "protein", "nucleotide", "refseq"]
```

A RefSeq category goes to a RefSeq downloader. Every other category in that list goes to a generic downloader, which
substitutes the version into the URL and filename, downloads the file and verifies it.

### Adding a database without touching code

If you add an entry under one of those four categories, with a URL, a filename and a database key, it is picked up
automatically. A new RefSeq entry needs its download method and the organism and type fields. That part really is
declarative, and it is the reason I wanted the file in the first place.

### Where it is not declarative

A new category needs code, because the install routine only fetches the four categories in its list. An entry under
any other name is silently never downloaded. I found this by reading the loop, not from a failure, which is the worst
kind of gap.

Software is also not in the file. Tools listed under software are mapped to environment keys by a dictionary in the
install code, so a new tool is a code edit. Kraken2, Centrifuge and Kaiju have entries in the file, but each still calls
its own hard-coded function, and I have not checked that a new Kaiju entry works from start to finish.

There are also keys nothing reads. A `version_template` key sits in the file, and no Python code uses it. And one path
ignores the file altogether: the Centrifuge prebuilt registration uses a fixed directory, and the lookup that would read
the path from the file is commented out.

### What I would do next

Move the category list and the software mapping into the file too, so the loader decides what to fetch from the file
alone. Delete keys nothing reads. And add a test that loads every entry and checks that its category is handled, so a
silent skip becomes a failing test.

The download side of this, which is where the integrity checks and locking live, is in
[the database download post]({% post_url 2026-03-20-downloading-databases-reliably %}).
