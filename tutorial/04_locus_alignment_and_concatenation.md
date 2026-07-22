# Part 4: Locus Alignment and Concatenation

## Inputs

- One unaligned multi-sample `.FNA` file per recovered gene in `work/recovered_genes/`.

## Outputs

- One aligned FASTA per gene and an alignment summary in `work/alignments/`.
- `work/concatenated/angiosperms353_tutorial.fasta`.
- `work/concatenated/angiosperms353_tutorial_partitions.txt`.
- `work/concatenated/angiosperms353_tutorial_summary.tsv`.

Part 4 will:

1. align each gene separately with [MAFFT](https://mafft.cbrc.jp/alignment/software/);
2. summarize the alignments with [AMAS](https://github.com/marekborowiec/AMAS); and
3. concatenate the aligned genes into one phylogenetic data matrix.

This quick tutorial retains all successfully aligned loci. It does not introduce automatic trimming or dataset-specific filtering thresholds.

## 1. Open the project and activate the environment

Open Terminal and enter:

```bash
cd "/Users/gauravmahajan/Documents/phylogeny"
conda activate angiosperms353
```

Confirm that MAFFT and AMAS are available:

```bash
mafft --version
AMAS.py --help
```

MAFFT is installed with the HybPiper environment. If `AMAS.py` reports `command not found`, install it into the active environment:

```bash
conda install -c conda-forge -c bioconda amas=1.0
```

Run all remaining commands from the `phylogeny/` project directory.

## 2. Understand why alignment comes first

The sequences in one recovered-gene FASTA represent the same gene, but they may have different lengths. Differences can result from insertions, deletions, or incomplete gene recovery.

An alignment inserts gap characters (`-`) so that bases inferred to share the same evolutionary position occur in the same column.

For example:

```text
Before alignment:
sample_A  ATGCTAGCTA
sample_B  ATGCTGCTA

After alignment:
sample_A  ATGCTAGCTA
sample_B  ATGCT-GCTA
```

A gap is not automatically a sequencing error. It may represent a biological insertion or deletion, or a position not recovered in one sample.

Each gene must be aligned separately because columns can be compared only among sequences from the same gene. The alignments can be joined after this positional correspondence has been established.

## 3. Create the output directories

Create one directory for individual gene alignments and another for the concatenated matrix:

```bash
mkdir -p work/alignments
mkdir -p work/concatenated
```

The original `.FNA` files in `work/recovered_genes/` will not be edited.

## 4. Align every recovered gene with MAFFT

Run MAFFT once for each recovered-gene FASTA:

```bash
for input in work/recovered_genes/*.FNA
do
  locus=$(basename "$input" .FNA)

  sed '/^>/ s/ .*$//' "$input" |
    mafft --auto - \
      > "work/alignments/${locus}.fasta"
done
```

For every input file, the loop:

1. obtains the locus name from the filename;
2. removes HybPiper's annotation after each sample name;
3. asks MAFFT to choose an appropriate alignment strategy with `--auto`; and
4. writes the aligned sequences to `work/alignments/`.

A recovered HybPiper header may look like:

```text
>GenusA_speciesA_CAP multi_hit_stitched_contig_comprising_2_hits
```

The text after the space describes how HybPiper recovered that sequence. It is useful diagnostic information, but it is not part of the sample name. The `sed` command changes the header passed to MAFFT to:

```text
>GenusA_speciesA_CAP
```

This matters because AMAS matches samples by their complete FASTA headers. Without this step, annotation differences among genes would be mistaken for different samples. The original `.FNA` files are not changed.

The `-` after `mafft --auto` tells MAFFT to read the header-cleaned sequences passed through the pipe.

MAFFT prints progress messages in Terminal. The aligned DNA sequences themselves are sent to the output FASTA by `>`.

List the alignments:

```bash
ls work/alignments
```

The number of aligned files should equal the number of recovered-gene files:

```bash
find work/recovered_genes -type f -name "*.FNA" | wc -l
find work/alignments -type f -name "*.fasta" | wc -l
```

With the supplied example, the first command normally reports nine recovered loci, so the second should report nine alignments.

Confirm that one alignment contains the four exact sample names:

```bash
grep "^>" work/alignments/4471.fasta
```

The result should be:

```text
>GenusA_speciesA_CAP
>GenusB_speciesB_CAP
>GenusC_speciesC_CAP
>GenusD_speciesD_CAP
```

## 5. Examine one alignment

Display the beginning of the alignment for locus `4471`:

```bash
head -n 12 work/alignments/4471.fasta
```

The file remains in FASTA format:

```text
>sample name
aligned DNA sequence
```

The important difference from the Part 3 FASTA is that the sequences are now aligned. Every sequence within this file has the same alignment length, including any `-` gap characters.

Do not compare column 100 of one gene with column 100 of another gene. The column positions are meaningful only within an individual locus alignment.

## 6. Summarize all alignments

Use AMAS to confirm that the FASTA files are aligned and create one summary table:

```bash
AMAS.py summary \
  -i work/alignments/*.fasta \
  -f fasta \
  -d dna \
  --check-align \
  -o work/alignments/alignment_summary.tsv
```

The arguments mean:

- `summary` requests alignment statistics;
- `-i` supplies all aligned locus files;
- `-f fasta` identifies their file format;
- `-d dna` identifies nucleotide data;
- `--check-align` checks that all sequences within each locus have equal length; and
- `-o` names the summary table.

Inspect the table:

```bash
head work/alignments/alignment_summary.tsv
```

Useful quantities include:

- **number of taxa** — how many samples are present in the locus;
- **alignment length** — the number of aligned nucleotide columns;
- **missing percent** — the proportion represented by gaps or undetermined bases;
- **variable sites** — columns containing more than one nucleotide state; and
- **parsimony-informative sites** — variable columns in which at least two states are each found in at least two samples.

With four samples, a parsimony-informative site usually has a two-sample versus two-sample pattern. These sites can help distinguish alternative groupings, although phylogenetic programs use the complete alignment rather than only these sites.

For this first tutorial, retain all the successfully aligned loci. We are not trimming gapped columns or applying a missing-data cutoff. End gaps can simply reflect partial gene recovery, and choosing defensible filters requires examining the full dataset.

## 7. Concatenate the locus alignments

Concatenation joins the gene alignments end to end for each sample:

```text
sample_A: gene 4471 + gene 4527 + gene 4691 + ...
sample_B: gene 4471 + gene 4527 + gene 4691 + ...
```

Run:

```bash
AMAS.py concat \
  -i work/alignments/*.fasta \
  -f fasta \
  -d dna \
  -u fasta \
  -t work/concatenated/angiosperms353_tutorial.fasta \
  -p work/concatenated/angiosperms353_tutorial_partitions.txt \
  --part-format raxml
```

AMAS creates:

- `angiosperms353_tutorial.fasta` — one long aligned DNA sequence per sample; and
- `angiosperms353_tutorial_partitions.txt` — the column range occupied by each gene.

Sample names must be identical among the locus files. The header-cleaning step above ensures that they are. AMAS uses those names to join the correct sequences. If a sample is absent from a locus, AMAS fills that part of its concatenated sequence with missing-data characters.

The partition file will resemble:

```text
DNA, 4471 = 1-...
DNA, 4527 = ...-...
```

These ranges preserve the boundary of each gene after concatenation. Part 5 will give the partition file to IQ-TREE so the genes do not have to be treated as one undivided sequence.

Display the actual partition ranges:

```bash
cat work/concatenated/angiosperms353_tutorial_partitions.txt
```

## 8. Summarize the concatenated matrix

Create a final matrix summary:

```bash
AMAS.py summary \
  -i work/concatenated/angiosperms353_tutorial.fasta \
  -f fasta \
  -d dna \
  --check-align \
  -o work/concatenated/angiosperms353_tutorial_summary.tsv
```

Inspect the result:

```bash
cat work/concatenated/angiosperms353_tutorial_summary.tsv
```

This reports the total matrix length, missing-data percentage, variable sites, and parsimony-informative sites across all included genes.

Confirm that the matrix contains four sample records:

```bash
grep -c "^>" work/concatenated/angiosperms353_tutorial.fasta
```

The expected result is:

```text
4
```

## 9. End of Part 4

The new working structure should resemble:

```text
phylogeny/
├── work/
│   ├── recovered_genes/
│   │   └── one unaligned .FNA file per recovered locus
│   ├── alignments/
│   │   ├── one aligned .fasta file per recovered locus
│   │   └── alignment_summary.tsv
│   └── concatenated/
│       ├── angiosperms353_tutorial.fasta
│       ├── angiosperms353_tutorial_partitions.txt
│       └── angiosperms353_tutorial_summary.tsv
└── tutorial/
    └── 04_locus_alignment_and_concatenation.md
```

Part 5 will use the concatenated FASTA and partition file to infer and examine a phylogenetic tree with IQ-TREE.
