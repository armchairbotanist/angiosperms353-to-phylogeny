# Reading a fastp HTML Report

This lecture is a guided reading of:

```text
work/fastp_reports/GenusB_speciesB_CAP.fastp.html
```

It was produced in Part 2 from one pair of Angiosperms353 example FASTQ files.

## Learning goals

By the end, you should be able to explain:

- what Angiosperms353 sequencing reads represent;
- the difference between a DNA insert, a read, a FASTQ record, and a read pair;
- how read length differs from insert size;
- why every called base has a quality score;
- what `before filtering` and `after filtering` mean; and
- what Q20, Q30, and GC content measure.

## 1. From plant DNA to an Angiosperms353 FASTQ file

### Angiosperms353 targets genes rather than the entire genome

A flowering-plant genome contains far more DNA than is needed for a first phylogenetic analysis. Angiosperms353 uses synthetic probes, often called baits, designed to capture 353 low-copy nuclear protein-coding genes across flowering plants.

A simplified laboratory workflow is:

1. extract genomic DNA from plant tissue;
2. break the DNA into shorter fragments;
3. ligate synthetic adapter molecules to the fragment ends;
4. hybridize the library to Angiosperms353 probes;
5. retain fragments that bind the probes; and
6. sequence the enriched library.


### Goal for fastp

An Illumina sequencer identifies each base from a fluorescent signal. Sometimes the signal is not clear enough to identify a base confidently. The sequencer records this uncertainty as a quality score.

fastp uses these quality scores and other information to evaluate the reads. It trims adapter sequence and removes read pairs that fail the selected filters.

> After fastp, the output contains reads that are mostly high quality and free of detected adapter sequence.

## 2. DNA inserts, reads, FASTQ records, and read pairs

### The DNA insert

During library preparation, laboratory adapters are attached to both ends of a plant-DNA fragment. The plant-derived sequence between the adapters is called the **insert**, or **library fragment**:

```text
adapter ── plant DNA insert ── adapter
```

### A sequencing read

During sequencing, the sequencer begins at one end of an insert. A **read** is the sequence of bases recorded as the sequencer moves along that DNA. For example:

```text
ACGTTGCA...
```

A read is usually only a short observation of the original DNA insert. One FASTQ record stores one read and its base-quality scores.

### What a FASTQ record contains

A FASTQ file contains many records. Each record stores one read using four lines:

```text
@read_identifier
ACGTTGCA
+
Q40,Q40,Q20,Q30,Q40,Q40,Q40,Q40
```

The four lines contain:

1. `@read_identifier` — the name of the read;
2. `ACGTTGCA` — the called DNA sequence;
3. `+` — a separator; and
4. the quality line — one quality score for every called base.

An `N` means the instrument could not confidently choose A, C, G, or T.

The quality scores are shown as Q20, Q30, and Q40 here for clarity. A real FASTQ file stores them using a character encoding, but that encoding is not important for reading the fastp report. The Phred scores are explained in Section 3.

### Paired-end sequencing

This library was sequenced from both ends:

```text
R1  ───────────────────────►
     plant DNA insert
     ◄───────────────────────  R2
```

The sequence read from one end is read 1, or R1. The sequence from the opposite end is read 2, or R2. Together they form one read pair derived from one DNA insert.

Paired-end data are stored in two coordinated FASTQ files. The R1 file contains one R1 record from every sequenced insert. The R2 file contains the corresponding R2 records in the same order:

```text
R1 record 1  pairs with  R2 record 1
R1 record 2  pairs with  R2 record 2
R1 record 3  pairs with  R2 record 3
```

Before filtering, this example has 298,627 records in the R1 file and 298,627 matching records in the R2 file. fastp therefore reports 597,254 read sequences, or 298,627 read pairs.

The report says:

```text
paired end (151 cycles + 151 cycles)
```

This means the instrument performed 151 base-calling cycles from the R1 end and another 151 cycles from the R2 end.

### Mean read length

The **mean read length** is the average number of bases present in each read:

```text
mean read length = total bases in the reads / number of reads
```

fastp reports the mean separately for R1 and R2. Before filtering, both means are 151 bp because every raw read contains all 151 sequencing cycles. After filtering, both means are 145 bp because fastp trimmed adapter sequence from some reads.

Mean read length is not insert size. Read length describes how many bases were read from one end of an insert; insert size describes the length of the plant-DNA fragment between the adapters. In fact, one DNA insert can actually lead to many reads.

## 3. Phred quality scores

A quality score is a prediction of the probability that a base call is wrong. Illumina reports these using the Phred scale:

```text
Q = -10 × log10(probability of an incorrect base call)
```

Useful landmarks are:

| Score | Estimated error probability | Plain-language interpretation |
|---|---:|---|
| Q10 | 1 in 10 | poor |
| Q20 | 1 in 100 | usually usable |
| Q30 | 1 in 1,000 | high quality |
| Q40 | 1 in 10,000 | very high quality |

A report value such as “94% Q30 bases” means approximately 94% of all called bases have an estimated error probability no greater than 1 in 1,000.

## 4. Open the worked report

From the project directory, run:

```bash
open work/fastp_reports/GenusB_speciesB_CAP.fastp.html
```

## 5. The report at a glance

We explain the most important values for this sample below:

| Measurement | Before filtering | After filtering |
|---|---:|---:|
| Read sequences | 597,254 | 591,746 |
| Read pairs | 298,627 | 295,873 |
| Total bases | 90,185,354 | 86,250,192 |
| Q20 bases | 97.34% | 97.80% |
| Q30 bases | 93.79% | 94.53% |
| Mean R1 length | 151 bp | 145 bp |
| Mean R2 length | 151 bp | 145 bp |
| GC content | 42.14% | 41.78% |

### Total reads counts both members of each pair

The report lists 597,254 reads before filtering. This number includes R1 and R2 separately.

Because the data are paired:

```text
597,254 read sequences ÷ 2 = 298,627 read pairs
```

After filtering:

```text
591,746 read sequences ÷ 2 = 295,873 read pairs
```

### Mean length changed from 151 bp to 145 bp

Before cleaning, R1 and R2 both have a mean length of 151 bp. After cleaning, both means are 145 bp.

A modest decrease is expected when detected adapter sequence is removed.

### Detected adapters

The report identifies standard Illumina TruSeq adapter sequences for both read directions. These are synthetic library molecules, not plant DNA.

Their detection is evidence that `--detect_adapter_for_pe` worked as intended.

Adapter sequence must be removed because it is laboratory sequence, not plant DNA, and can interfere with mapping and assembly.

### Before filtering

“Before filtering” describes the original FASTQ input.

For this sample:

- 93.79% of all bases were Q30 or higher;
- 97.34% were Q20 or higher; and
- GC content was 42.14%.

These values combine R1 and R2. The separate plots later in the report show that R1 was slightly better than R2.

GC content is the percentage of called bases that are G or C. In target-capture data, it reflects the DNA regions captured and should not be interpreted as the GC content of the entire plant genome.

### After filtering

“After filtering” describes the reads written to `work/clean_reads/`.

After fastp:

- Q30 increased from 93.79% to 94.53%;
- Q20 increased from 97.34% to 97.80%;
- GC content changed slightly from 42.14% to 41.78%; and
- 99.08% of reads remained.

The improvement in quality percentages is expected because low-quality pairs and non-biological adapter bases were removed.

Overall, fastp retained nearly all read pairs while improving base-quality percentages and removing adapter sequence. These cleaned reads are suitable for the HybPiper tutorial.
