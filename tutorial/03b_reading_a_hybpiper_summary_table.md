# Reading a HybPiper Summary Table

This lesson is a guided reading of:

```text
work/hybpiper/hybpiper_stats.tsv
```

The table was produced in Part 3 by running `hybpiper stats` after the four example samples were assembled against the 10-gene tutorial target.

The intended reader understands basic genetics and plant biology but does not need previous experience with sequence capture, read mapping, genome assembly, or bioinformatics.

## Learning goals

By the end, you should be able to explain:

- why each row represents one biological sample;
- how reads progress from mapping to an extracted gene sequence;
- the difference between a read, contig, and recovered sequence;
- what the gene-length thresholds measure;
- what paralog, stitched-contig, and chimera warnings mean;
- which values are most useful when comparing samples; and
- what the example table says about the four plants.

## 1. What question does this table answer?

The fastp report from the previous lesson asked:

> Are the raw sequencing reads technically usable?

The HybPiper summary asks a later biological and computational question:

> Did reads matching the target genes assemble into useful coding sequences?

A sample can have excellent read quality but weak target-gene recovery. Conversely, a sample can have fewer reads yet recover many useful genes. The fastp and HybPiper summaries therefore describe different stages of the experiment.

The file is a tab-separated value, or TSV, table:

- each **row** is one sample;
- each **column** is one measurement; and
- tabs separate the values.

## 2. Open the table

From the project directory, display the table in aligned columns:

```bash
column -t -s $'\t' work/hybpiper/hybpiper_stats.tsv | less -S
```

Use the arrow keys to move through the table and press `q` to leave `less`.

The table is wide because it summarizes several stages of HybPiper. It is easier to understand those columns as a recovery funnel:

```text
input reads
    |
    v
reads mapped to a target gene
    |
    v
genes with mapped reads
    |
    v
genes with assembled contigs
    |
    v
genes with extracted coding sequences
    |
    v
genes with substantially complete sequences
```

At each stage, information can be lost. A few reads matching a gene may not be sufficient for assembly, and an assembled contig may not contain a recognizable coding sequence.

## 3. The example results at a glance

The main recovery measurements are:

| Sample | NumReads | ReadsMapped | PctOnTarget | GenesMapped | GenesWithContigs | GenesWithSeqs |
|---|---:|---:|---:|---:|---:|---:|
| GenusA_speciesA_CAP | 720,076 | 12,118 | 1.7% | 10 | 9 | 9 |
| GenusB_speciesB_CAP | 591,746 | 10,082 | 1.7% | 10 | 9 | 9 |
| GenusC_speciesC_CAP | 372,910 | 2,343 | 0.6% | 9 | 9 | 9 |
| GenusD_speciesD_CAP | 412,720 | 7,113 | 1.7% | 10 | 9 | 9 |

The recovered-length measurements are:

| Sample | GenesAt25pct | GenesAt50pct | GenesAt75pct | GenesAt150pct | TotalBasesRecovered |
|---|---:|---:|---:|---:|---:|
| GenusA_speciesA_CAP | 8 | 7 | 6 | 0 | 5,007 |
| GenusB_speciesB_CAP | 8 | 6 | 5 | 0 | 4,998 |
| GenusC_speciesC_CAP | 8 | 4 | 2 | 0 | 3,585 |
| GenusD_speciesD_CAP | 9 | 6 | 4 | 0 | 5,187 |

All four samples produced sequences for nine genes. The principal difference is completeness: sample C recovered shorter portions of those genes than the other samples.

## 4. Sample and read columns

### `Name`

`Name` is the sample identifier supplied to HybPiper with `--prefix`.

In this tutorial, names such as `GenusA_speciesA_CAP` follow the same samples from the FASTQ filenames through cleaning, assembly, alignment, and tree construction. Keeping the identifier unchanged prevents sequences from different individuals from being confused.

### `NumReads`

`NumReads` is the total number of input read sequences supplied to HybPiper.

R1 and R2 are counted separately. Therefore:

```text
720,076 reads = 360,038 read pairs
```

These are the cleaned reads written by fastp, not the number of genes or DNA fragments recovered.

More reads can improve recovery, but read count alone is not a measure of success. Many reads may be off target, duplicates, or concentrated on only a few genes.

### `ReadsMapped`

`ReadsMapped` is the number of input read sequences that matched sequences in the target FASTA.

Mapping is an initial similarity search. A mapped read is evidence that part of the sample resembles a target gene, but a single short read is not yet a recovered gene.

### `PctOnTarget`

`PctOnTarget` is calculated as:

```text
100 x ReadsMapped / NumReads
```

For sample B:

```text
100 x 10,082 / 591,746 = approximately 1.7%
```

This tutorial uses only 10 of the 353 Angiosperms353 genes. Reads matching the other 343 genes are not counted as on target here, even though they may be genuine Angiosperms353 reads. The values of 0.6% to 1.7% therefore should not be compared directly with a full 353-gene analysis.

Within this tutorial, sample C has the lowest on-target percentage. That is useful as a relative comparison among samples processed with the same target file.

## 5. The gene-recovery funnel

### `GenesMapped`

`GenesMapped` is the number of target genes that received mapped reads.

The tutorial target contains 10 genes, so the maximum is 10. Samples A, B, and D had mapped reads for all 10. Sample C had mapped reads for nine.

This column asks whether HybPiper found evidence for a gene. It does not say that a complete sequence was assembled.

### `GenesWithContigs`

A **contig** is a longer sequence assembled by finding overlaps among many short reads.

`GenesWithContigs` counts genes for which SPAdes produced contigs that could proceed to the coding-sequence extraction stage.

If `GenesMapped` is larger than `GenesWithContigs`, some genes had matching reads but not enough consistent information for a successful assembly. Low read depth, uneven capture, sequencing errors, or repetitive sequence can all contribute.

### `GenesWithSeqs`

`GenesWithSeqs` counts genes for which HybPiper extracted a coding DNA sequence from the assembled contigs.

This is the most direct count of genes available for later alignment. All four samples have nine recovered sequences.

The three columns form a useful progression:

```text
GenesMapped >= GenesWithContigs >= GenesWithSeqs
```

For sample A:

```text
10 genes had mapped reads
 9 genes produced contigs
 9 genes produced extracted sequences
```

Mapping succeeded for one gene that did not assemble. The other nine passed from assembly to sequence extraction.

## 6. How complete are the recovered genes?

Angiosperms353 provides multiple reference sequences for many genes. HybPiper calculates the mean reference length for each gene and compares the recovered sequence with that value.

The length counts exclude `N` characters. An `N` marks an unknown nucleotide, so a stretch of Ns does not increase the reported recovered length.

The four threshold columns are **cumulative**, not separate categories. A gene that is 80% of its mean target length is counted in:

- `GenesAt25pct`;
- `GenesAt50pct`; and
- `GenesAt75pct`.

### `GenesAt25pct`

This is the number of recovered genes longer than 25% of their mean target length.

A gene below this threshold is very fragmentary. It may still be a real sequence, but it contains little of the expected coding region.

### `GenesAt50pct`

This is the number of recovered genes longer than 50% of their mean target length.

These genes contain more than half of the expected coding sequence and are usually more informative than very short fragments.

### `GenesAt75pct`

This is the number of recovered genes longer than 75% of their mean target length.

This is a convenient indicator of relatively complete recovery. It is not a universal pass/fail rule: genuine genes can differ in length among species, and the target mean may not perfectly represent every lineage.

### `GenesAt150pct`

This is the number of recovered genes longer than 150% of their mean target length.

An unusually long sequence deserves inspection. Possible explanations include:

- a genuinely longer gene in the sampled lineage;
- incorrect joining of contigs;
- an assembly problem;
- a duplicated gene copy; or
- a target reference that is unusually short.

A nonzero value is a warning to investigate, not proof that the sequence is wrong. All four tutorial samples have zero.

### Reading the thresholds together

Sample A has:

```text
GenesWithSeqs = 9
GenesAt25pct  = 8
GenesAt50pct  = 7
GenesAt75pct  = 6
```

This means:

- six genes are longer than 75% of the target mean;
- one additional gene is between 50% and 75%;
- one additional gene is between 25% and 50%; and
- one recovered gene is no more than 25%.

Sample C also has nine sequences, but only two are longer than 75% of the target mean. Its gene count looks good while its sequence completeness is comparatively weak. This is why `GenesWithSeqs` should not be interpreted alone.

## 7. Potential paralogs

### Orthologs and paralogs

Two homologous genes are **orthologs** when their history is primarily separated by a speciation event. Phylogenetic studies usually aim to compare orthologous copies across species.

They are **paralogs** when they arose through gene duplication. Plants frequently experience gene duplication and whole-genome duplication, so even a low-copy target set can encounter paralogs.

If different samples contribute different duplicated copies, a gene tree may reflect duplication history rather than species history.

### `ParalogWarningsLong`

This counts genes for which HybPiper found more than one plausible coding sequence on different contigs, with each covering more than 75% of the selected target reference by default.

Two long alternatives suggest that the sample may contain multiple copies of the gene.

### `ParalogWarningsDepth`

This counts genes for which contig coverage across much of the target supports more than one possible sequence copy.

It is a second way of detecting possible duplicated copies, based on the depth pattern of assembled contigs rather than only the presence of multiple long sequences.

### How to interpret a warning

A paralog warning is not automatic proof of gene duplication. Allelic variation, assembly fragmentation, contamination, or repetitive sequence can sometimes produce a similar signal.

The correct response is to mark the gene for later inspection. Useful evidence can include:

- whether the warning occurs in one or many samples;
- the alignment;
- the alternative sequences recovered by HybPiper; and
- the individual gene tree.

Both paralog-warning columns are zero for every sample in this tutorial.

## 8. Stitched contigs and chimera warnings

Sometimes one SPAdes contig covers the entire recovered coding region. In other cases, separate contigs match different non-overlapping portions of the same reference gene. HybPiper can join those portions into a **stitched contig**.

### `GenesWithoutStitchedContigs`

This counts genes whose recovered sequence came from a single SPAdes contig. No stitching was needed.

### `GenesWithStitchedContigs`

This counts genes whose recovered sequence combined matching regions from multiple SPAdes contigs.

Stitching is not automatically bad. It can recover a more complete gene when the read assembly is fragmented. However, an alignment containing unusual gaps or conflicting segments deserves inspection.

For each tutorial sample:

```text
GenesWithoutStitchedContigs + GenesWithStitchedContigs = GenesWithSeqs
```

For sample A:

```text
2 + 7 = 9
```

### `GenesWithStitchedContigsSkipped`

This counts genes for which stitching was deliberately disabled with HybPiper's `--no_stitched_contig` option. In that situation, HybPiper retains only the longest matching hit.

This is unrelated to the tutorial's `--no_intronerate` option. The latter skips recovery of introns and supercontigs; it does not disable stitching of coding sequences.

The tutorial did not disable stitching, so every value is zero.

### `GenesWithChimeraWarning`

A chimeric stitched sequence incorrectly combines pieces that do not belong to the same gene copy. HybPiper can perform an optional read-pair check for this problem.

This column counts genes flagged by that check. The Part 3 command did not request the optional chimera test, so zeros here mean that no warnings were generated; they do not mean that every stitched sequence was independently proven non-chimeric.

## 9. `TotalBasesRecovered`

`TotalBasesRecovered` is the total number of A, C, G, and T nucleotides across all extracted gene sequences for one sample. `N` characters are excluded.

This measurement combines gene number and gene completeness:

- recovering more genes usually increases the total;
- recovering longer portions of genes increases the total; and
- long stretches of unknown bases do not inflate it.

Sample D has the largest total, 5,187 bases. Sample C has the smallest, 3,585 bases, consistent with its lower `GenesAt50pct` and `GenesAt75pct` counts.

A high total should still be interpreted alongside `GenesAt150pct` and paralog warnings. An incorrectly long sequence could otherwise make the total look better.

## 10. What does the tutorial table tell us?

The broad conclusions are:

1. **All four samples are represented.** No sample failed completely.
2. **Nine of the ten tutorial genes were recovered for every sample.**
3. **Gene 4744 was not recovered in any sample.** This explains why Part 3 produced nine `.FNA` files rather than ten.
4. **Sample C is the weakest sample.** It has the lowest on-target percentage, the fewest genes longer than 75% of their targets, and the lowest total recovered length.
5. **Sample C is still useful for this tutorial.** It has a sequence for each of the same nine genes as the other samples.
6. **No paralog or excessive-length warnings were reported.**
7. **Many recovered genes were stitched.** This makes alignment inspection in Part 4 worthwhile, but it is not by itself a reason to remove them.

The table supports continuing with all four samples and all nine recovered genes.

## 11. Related HybPiper tables

The summary table answers sample-level questions. Two companion tables provide gene-level detail.

### `seq_lengths.tsv`

This table contains:

- one column per target gene;
- a `MeanLength` row containing each gene's mean reference length; and
- one row per sample containing the recovered length for every gene.

A zero means no sequence was recovered. This table shows that gene 4744 has a zero for all four samples.

### `gene_read_counts_all.tsv`

This table reports how many reads were assigned to each gene in each sample.

It can help explain why one gene failed or why one sample recovered shorter sequences. Read count is supporting evidence, not a guarantee of correct assembly.

## 12. A practical reading order

When examining a new HybPiper summary, ask:

1. Are all expected samples present?
2. Does one sample have far fewer `GenesWithSeqs` than the others?
3. Does one sample have much lower `GenesAt75pct` or `TotalBasesRecovered`?
4. Are there any `GenesAt150pct`, paralog warnings, or chimera warnings?
5. Do the stitched-contig counts suggest that alignment inspection will be especially important?
6. Can an unusual summary value be explained using `seq_lengths.tsv`, `gene_read_counts_all.tsv`, or the fastp report?

Avoid declaring a sample good or bad from one universal cutoff. Comparisons among samples prepared and sequenced together are usually more informative.

## 13. Main lesson

The most useful interpretation is not simply:

> How many genes were recovered?

It is:

> How many genes were recovered, how complete are they, and are there warnings that could indicate the wrong gene copy or an unreliable assembly?

For this example, nine genes can proceed to alignment. Sample C contributes shorter sequences, so it should receive extra attention when the alignments are inspected.

Column definitions in this lesson follow HybPiper 2.3.4 and the [official HybPiper `stats` documentation](https://github.com/mossmatters/HybPiper/wiki#hybpiper-stats).
