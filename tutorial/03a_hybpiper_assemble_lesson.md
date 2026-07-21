# Lesson: Understanding `hybpiper assemble`

## Learning goal

By the end of this lesson, you should understand the three main stages of `hybpiper assemble`, why each sample is processed independently, and how the output is organized.

---

## 1. Inputs and purpose

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

The target file tells HybPiper which genes it should try to recover. The prefix names the current sample and its output.

HybPiper processes one sample in each `assemble` run. It does not combine reads from different individuals during assembly.

---

### What problem is being solved?

The FASTQ files contain reads from many genomic regions. HybPiper must first determine which reads probably belong to each target gene.

For example, it needs to separate the reads conceptually into groups such as:

```text
Reads matching gene 1
Reads matching gene 2
Reads matching gene 3
...
```

## 2. The three main stages

The `assemble` command solves this problem by breaking it into three broad tasks:

1. Map reads to target loci.
2. Assemble the reads for each locus.
3. Identify and extract the coding sequence.

---

### Stage 1: Map reads to targets with BWA

In the first stage, HybPiper uses **BWA** to compare the sequencing reads with the gene sequences in the Angiosperms353 target FASTA file.

BWA asks:

> Which target sequence does this read resemble?

A read that resembles target gene 1 is assigned to the group for gene 1. A read that resembles target gene 2 is assigned to the group for gene 2.

At this stage, the reads are being mapped and grouped by target gene. BWA does not reconstruct the complete gene.

#### Important point

Mapping to a target does not mean that your sample is assumed to be identical to the reference. The target sequence is a guide for finding relevant reads; the recovered sequence is assembled from the sample's own reads.

---

### Stage 2: Assemble each locus with SPAdes

#### Why assembly is needed

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

#### What SPAdes does

HybPiper sends the reads assigned to each locus to **SPAdes**.

SPAdes assembles the reads into longer sequences called **contigs**.

Each target locus is assembled separately:

```text
Reads for gene 1 → contigs for gene 1
Reads for gene 2 → contigs for gene 2
Reads for gene 3 → contigs for gene 3
```

This prevents reads from unrelated genes from being assembled together.

#### Possible outcomes

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

### Stage 3: Extract the coding sequence with Exonerate

#### Why another step is needed

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

#### What Exonerate does

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

#### Supercontigs

HybPiper can also retain the broader sequence containing:

- coding exons,
- introns,
- flanking regions.

This combined sequence is often called a **supercontig**.

Supercontigs may contain more variation than exons alone, but they can also be harder to align reliably. Part 3 uses `--no_intronerate`, so the quick tutorial skips supercontig recovery.

---

## 3. What the output directory represents

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
├── paralog warnings
└── log files
```

The exact directory structure depends on the HybPiper version and options used.

The important idea is:

> One HybPiper directory represents one sample, while the files inside it represent the different target loci recovered from that sample.

---

## 4. What happens after `assemble`

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

Part 3 uses `hybpiper retrieve_sequences` to perform this collection step. Part 4 will align the sequences and prepare them for phylogenetic analysis.

---
