# Part 3: Angiosperms353 Locus Recovery with HybPiper

Part 2 produced:

- eight cleaned FASTQ files;
- a list containing the four sample names; and
- a 10-gene tutorial target FASTA.

Part 3 will:

1. recover targeted coding sequences with [HybPiper](https://github.com/mossmatters/HybPiper/wiki/Tutorial);
2. summarize how many genes were recovered; and
3. collect one unaligned FASTA file per gene for Part 4.

## 1. Open the project and activate the environment

Open Terminal and enter:

```bash
cd "/Users/gauravmahajan/Documents/phylogeny"
conda activate angiosperms353
```

Confirm that HybPiper is available:

```bash
hybpiper --version
```

Create the HybPiper working directory:

```bash
mkdir -p work/hybpiper
```

Run all remaining commands from the `phylogeny/` project directory unless a step says otherwise.

## 2. Understand what HybPiper will do

HybPiper processes each sample independently. Its main `assemble` command performs three broad tasks:

1. **Map reads to targets.** BWA identifies reads matching the nucleotide target sequences.
2. **Assemble each locus.** SPAdes assembles the matching reads separately for every target locus.
3. **Extract coding sequences.** Exonerate identifies and joins the recovered coding regions.

For a more detailed explanation, read [Understanding `hybpiper assemble`](03a_hybpiper_assemble_lesson.md).

The output prefix becomes both the sample label in recovered FASTA files and the name of that sample's HybPiper directory.

This tutorial recovers coding sequences only. It skips intron and supercontig recovery and uses the 10-gene target created in Part 2.

## 3. Recover 10 Angiosperms353 genes with HybPiper

Run HybPiper once for each cleaned sample:

```bash
while read -r sample
do
  hybpiper assemble \
    -t_dna references/angiosperms353/Angiosperms353_first_10_genes.fasta \
    -r "work/clean_reads/${sample}_R1.clean.fq.gz" \
       "work/clean_reads/${sample}_R2.clean.fq.gz" \
    --prefix "${sample}" \
    --bwa \
    --no_intronerate \
    --hybpiper_output work/hybpiper
done < work/sample_names.txt
```

The important arguments are:

- `-t_dna` supplies the smaller 10-gene nucleotide target FASTA;
- `-r` supplies the cleaned paired reads;
- `--prefix` gives the sample its permanent identifier;
- `--bwa` tells HybPiper to map against nucleotide targets with BWA;
- `--no_intronerate` skips intron and supercontig recovery; and
- `--hybpiper_output` keeps all sample directories inside `work/hybpiper/`.

The loop runs samples sequentially, which is easier to follow and limits memory use. Within each sample, HybPiper can assemble several genes concurrently. The smaller 10-gene target makes this much quicker than assembling all 353 genes.

The command deliberately omits `--cpu`. HybPiper therefore detects the computer automatically and uses all available processor cores minus one. This leaves one core for macOS and allows the same command to work on Macs with different numbers of cores.

After completion, list the HybPiper directory:

```bash
ls work/hybpiper
```

You should see one directory named for each sample.

## 4. Summarize locus recovery

HybPiper provides `stats` to combine recovery information across samples.

Move into the directory containing the four sample-output directories:

```bash
cd work/hybpiper
```

Create the summary tables:

```bash
hybpiper stats \
  -t_dna ../../references/angiosperms353/Angiosperms353_first_10_genes.fasta \
  gene ../sample_names.txt \
  --hybpiper_dir . \
  --no_heatmap
```

The main output files are:

- `hybpiper_stats.tsv` — one row per sample with overall recovery statistics;
- `seq_lengths.tsv` — recovered sequence length for every sample and locus; and
- `gene_read_counts_all.tsv` — read counts for every sample and locus.

For a detailed explanation of every column and a worked interpretation of the four samples, read [Reading a HybPiper Summary Table](03b_reading_a_hybpiper_summary_table.md).

Inspect the overall table:

```bash
head -n 5 hybpiper_stats.tsv
```

Important columns include:

- `PctOnTarget` — percentage of input reads mapped to the target file;
- `GenesWithSeqs` — number of loci with an extracted coding sequence;
- `GenesAt75pct` — number recovered to more than 75% of mean target length;
- `ParalogWarningsLong` and `ParalogWarningsDepth` — possible duplicated-gene warnings; and
- `TotalBasesRecovered` — total recovered nucleotide length.

Do not expect every sample to recover all 10 tutorial genes. The useful comparison is whether one sample recovers far fewer or much shorter genes than the others. Paralog warnings are diagnostic flags; they should be examined later rather than treated as automatic proof that a sequence is unusable.

## 5. Retrieve one FASTA file per locus

The sequences are currently nested inside the four sample directories. `retrieve_sequences` reorganizes them into one multi-sample FASTA file per locus:

```bash
hybpiper retrieve_sequences dna \
  -t_dna ../../references/angiosperms353/Angiosperms353_first_10_genes.fasta \
  --sample_names ../sample_names.txt \
  --hybpiper_dir . \
  --fasta_dir ../recovered_genes
```

Return to the project directory:

```bash
cd ../..
```

List a few recovered locus files:

```bash
ls work/recovered_genes | head
```

Count them:

```bash
find work/recovered_genes -type f -name "*.FNA" | wc -l
```

There can be up to 10 `.FNA` files. Each file represents one tutorial gene and contains the recovered, unaligned coding sequence for every sample in which that gene was found.

A sample can therefore be absent from an individual locus file without being absent from the project.

Do not concatenate these files yet. Sequences from the same locus must first be aligned and examined in Part 4.

## 6. End of Part 3

The new working structure should resemble:

```text
phylogeny/
├── references/
│   └── angiosperms353/
│       ├── Angiosperms353_targetSequences.fasta
│       └── Angiosperms353_first_10_genes.fasta
├── work/
│   ├── sample_names.txt
│   ├── clean_reads/
│   │   └── eight cleaned FASTQ files
│   ├── fastp_reports/
│   │   └── four HTML and four JSON reports
│   ├── hybpiper/
│   │   ├── one directory per sample
│   │   ├── gene_read_counts_all.tsv
│   │   ├── hybpiper_stats.tsv
│   │   └── seq_lengths.tsv
│   └── recovered_genes/
│       └── one unaligned .FNA file per recovered locus
└── tutorial/
    ├── 01_environment_setup_and_data_retrieval.md
    ├── 02_read_cleaning_and_target_preparation.md
    ├── 02a_reading_a_fastp_html_report.md
    ├── 03_hybpiper_locus_recovery.md
    ├── 03a_hybpiper_assemble_lesson.md
    └── 03b_reading_a_hybpiper_summary_table.md
```

Part 4 will align the sequences within each locus, inspect and filter the alignments, then concatenate the retained loci into a phylogenetic data matrix.
