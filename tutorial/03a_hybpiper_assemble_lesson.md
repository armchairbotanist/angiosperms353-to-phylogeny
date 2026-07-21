# Lesson: Understanding `hybpiper assemble`

## Learning goal

By the end of this lesson, you should understand what HybPiper does when it runs the `assemble` command, why each sample is processed independently, and how the output is organized.

---

## 1. Where HybPiper fits in the pipeline

After target-capture sequencing, you usually receive paired-end FASTQ files for each sample:

```text
ECH001_R1.fastq.gz
ECH001_R2.fastq.gz
```

These files contain millions of short sequencing reads. The reads are not yet organized into genes. We usually expect many of these reads (say 100) to correspond to a single gene (many reads would overlap). We would like to therefore like to assemble these reads into how the gene looks like for the individual.

HybPiper reconstructs the targeted genes from these reads.

For Angiosperms353 data, the overall idea is:

```text
FASTQ reads
    ↓
HybPiper assemble
    ↓
Recovered gene sequences for one sample
    ↓
Collect the same genes across samples
    ↓
Alignment and phylogenetic analysis
```

We will do the first 3 steps here.

---

## 2. HybPiper processes one sample at a time

HybPiper runs independently for each individual.

For example:

```text
ECH001 reads → recovered genes for ECH001
ECH002 reads → recovered genes for ECH002
ECH003 reads → recovered genes for ECH003
```

It does not initially combine reads from different individuals.

This is important because each individual may have:

- different sequencing depth,
- different missing loci,
- different alleles,
- different levels of contamination,
- different paralog warnings.

After all samples have been processed, HybPiper can collect the same locus across individuals.

---

## 3. Inputs to `hybpiper assemble`

For one sample, the main inputs are:

1. The paired FASTQ files.
2. A target file containing reference sequences for the genes of interest.
3. A unique sample name or prefix.

Conceptually:

```text
Sample reads:
ECH001_R1.fastq.gz
ECH001_R2.fastq.gz

Target file:
Angiosperms353 reference genes

Sample prefix:
ECH001
```

The target file tells HybPiper which genes it should try to recover.

---

## 4. The three main stages

The `assemble` command performs three broad tasks:

1. Map reads to target loci.
2. Assemble the reads for each locus.
3. Identify and extract the coding sequence.

---

# Stage 1: Map reads to targets

## What problem is being solved?

The FASTQ files contain reads from many genomic regions. HybPiper must first determine which reads probably belong to each target gene.

For example, it needs to separate the reads conceptually into groups such as:

```text
Reads matching gene 1
Reads matching gene 2
Reads matching gene 3
...
```

## What BWA does

When nucleotide targets are used, HybPiper commonly uses **BWA** to compare the sequencing reads with the target sequences.

BWA asks:

> Which target sequence does this read resemble?

A read that resembles target gene 1 is assigned to the group for gene 1. A read that resembles target gene 2 is assigned to the group for gene 2.

At this stage, the reads are only being sorted. BWA does not reconstruct the complete gene.

## Important point

Mapping to a target does not mean that your sample is assumed to be identical to the reference.

The target sequence acts as a guide for finding relevant reads. The sequence recovered from your sample is assembled from your sample's own reads.

---

# Stage 2: Assemble each locus

## Why assembly is needed

Sequencing reads are short. A gene may be represented by hundreds or thousands of overlapping reads.

For example:

```text
Read 1: ATGCTAGCTA
Read 2:     TAGCTAGGCA
Read 3:          AGGCATCCA
```

Because the reads overlap, they can be combined into a longer sequence:

```text
ATGCTAGCTAGGCATCCA
```

## What SPAdes does

HybPiper sends the reads assigned to each locus to **SPAdes**.

SPAdes assembles the reads into longer sequences called **contigs**.

Each target locus is assembled separately:

```text
Reads for gene 1 → contigs for gene 1
Reads for gene 2 → contigs for gene 2
Reads for gene 3 → contigs for gene 3
```

This prevents reads from unrelated genes from being assembled together.

## Possible outcomes

For a given locus, HybPiper may recover:

- one long contig,
- several shorter contigs,
- only part of the gene,
- no useful contig,
- multiple plausible contigs.

Multiple plausible contigs can occur because of:

- gene duplication,
- divergent alleles,
- polyploidy,
- contamination,
- assembly errors.

These cases may produce paralog warnings later.

---

# Stage 3: Extract the coding sequence

## Why another step is needed

The assembled contigs may contain more than the coding sequence.

A contig may include:

```text
flanking region
exon
intron
exon
intron
exon
flanking region
```

For many phylogenetic analyses, HybPiper needs to identify the coding exons and join them in the correct order.

## What Exonerate does

**Exonerate** compares the assembled contigs with the reference target and identifies the regions corresponding to the coding sequence.

For a gene with three exons:

```text
Contig:
flank — exon 1 — intron — exon 2 — intron — exon 3 — flank
```

Exonerate extracts:

```text
exon 1 + exon 2 + exon 3
```

The result is the recovered coding sequence for that gene in that individual.

## Supercontigs

HybPiper can also retain the broader sequence containing:

- coding exons,
- introns,
- flanking regions.

This combined sequence is often called a **supercontig**.

Supercontigs may contain more variation than exons alone and can be useful for closely related species or populations. However, they can also be harder to align reliably.

---

## 5. What the sample prefix does

Suppose the sample prefix is:

```text
ECH001
```

The prefix has two main roles.

### It names the sample directory

HybPiper creates a directory for that sample:

```text
ECH001/
```

This directory contains intermediate files, logs, assemblies, and recovered sequences.

### It labels recovered sequences

The recovered FASTA sequence will use the prefix as its sequence name:

```fasta
>ECH001
ATGCTAGCTAGGCATCCA
```

Later, when the same gene is collected across several individuals, the FASTA file may look like:

```fasta
>ECH001
ATGCTAGCTAGGCATCCA

>ECH002
ATGCTAGATAGGCATCCA

>ECH003
ATGTTAGATAGGCATCCA
```

These labels later become the tip names in gene trees and species trees.

For this reason, prefixes should be:

- unique,
- stable,
- easy to match with the metadata table,
- free of spaces,
- consistent across the entire project.

---

## 6. What the output directory represents

The sample directory stores the results for one individual.

Conceptually:

```text
ECH001/
├── mapping results
├── reads separated by target
├── SPAdes assemblies
├── Exonerate results
├── recovered nucleotide sequences
├── recovered protein sequences
├── supercontigs
├── paralog warnings
└── log files
```

The exact directory structure depends on the HybPiper version and options used.

The important idea is:

> One HybPiper directory represents one sample, while the files inside it represent the different target loci recovered from that sample.

---

## 7. What happens after `assemble`

After running `hybpiper assemble` for every individual, the data are still organized by sample:

```text
ECH001 → many recovered loci
ECH002 → many recovered loci
ECH003 → many recovered loci
```

For phylogenetics, you need to reorganize the results by locus:

```text
gene_1.fasta → gene 1 from all samples
gene_2.fasta → gene 2 from all samples
gene_3.fasta → gene 3 from all samples
```

HybPiper's sequence-retrieval tools perform this collection step.

Each locus can then be:

1. aligned across samples,
2. checked for missing or suspicious sequences,
3. used to infer a gene tree.

The gene trees can then be combined into a species tree, for example with ASTRAL.

---

## 8. A complete mental model

You can think of the process as a library-sorting task.

### BWA: find the right books

BWA identifies which short reads probably belong to each target gene.

```text
Mixed reads → reads grouped by gene
```

### SPAdes: reconstruct the pages

SPAdes joins overlapping reads into longer contigs.

```text
Short overlapping reads → assembled contigs
```

### Exonerate: identify the relevant chapters

Exonerate identifies the coding exons and joins them in the correct order.

```text
Contigs containing exons and introns → recovered coding sequence
```

In one sentence:

> BWA finds which reads belong to each target gene, SPAdes assembles those reads into contigs, and Exonerate identifies and joins the coding regions.

---

## 9. Common misunderstandings

### “BWA produces the final gene sequence.”

No. BWA only identifies reads that resemble each target. SPAdes and Exonerate perform the reconstruction and extraction.

### “HybPiper combines all individuals during assembly.”

No. Each individual is assembled independently.

### “The recovered sequence is copied from the reference target.”

No. The target guides read assignment and gene identification. The recovered sequence comes from the sample's sequencing reads.

### “Every sample should recover all 353 loci.”

Not necessarily. Missing loci can result from poor DNA quality, low sequencing depth, inefficient capture, sequence divergence, or assembly failure.

### “One output sequence always means one true gene copy.”

Not necessarily. Duplications, alleles, polyploidy, contamination, and assembly artifacts can complicate the result.

### “The `assemble` command produces a phylogeny.”

No. It produces gene sequences for one sample. Alignment and tree inference happen later.

---

## 10. Check your understanding

1. Why does HybPiper process each sample independently?
2. What role does the target file play?
3. What is the difference between mapping and assembly?
4. Why are SPAdes assemblies processed with Exonerate?
5. What does the sample prefix control?
6. What is the difference between an exon sequence and a supercontig?
7. What must happen before recovered sequences can be used to infer a tree?

---

## Summary

The `hybpiper assemble` command converts raw sequencing reads from one sample into reconstructed target-gene sequences.

Its main stages are:

```text
BWA
Map reads to target loci
    ↓
SPAdes
Assemble the reads for each locus
    ↓
Exonerate
Identify and join coding regions
```

The sample prefix names the HybPiper directory and labels the recovered sequences. After all samples are processed independently, the same locus is collected across samples, aligned, and used for phylogenetic analysis.
