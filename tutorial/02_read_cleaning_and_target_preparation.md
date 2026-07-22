# Part 2: Read Cleaning and Tutorial Target Preparation

## Inputs

- Four raw FASTQ pairs in `raw_data/angiosperms353_example/`.
- `references/angiosperms353/Angiosperms353_targetSequences.fasta`.

## Outputs

- `work/sample_names.txt`.
- Eight cleaned FASTQ files in `work/clean_reads/`.
- Four HTML and four JSON reports in `work/fastp_reports/`.
- `references/angiosperms353/Angiosperms353_first_10_genes.fasta`.

Part 2 will:

1. clean the paired FASTQ reads with [fastp](https://github.com/OpenGene/fastp);
2. review one fastp quality report; and
3. make a smaller target file containing 10 of the 353 genes.

Part 3 will use the cleaned reads and the smaller target file to recover coding sequences with HybPiper.

Using 10 genes makes the assembly practical for a classroom exercise. The original FASTQ files and the downloaded 353-gene target FASTA will not be changed.

## 1. Open the project and activate the environment

Open Terminal and enter:

```bash
cd "/Users/gauravmahajan/Documents/phylogeny"
conda activate angiosperms353
```

Confirm that fastp is available:

```bash
fastp --version
```

Run all remaining commands from the `phylogeny/` project directory.

## 2. Create working directories

Create separate directories for cleaned reads and quality reports:

```bash
mkdir -p work/clean_reads
mkdir -p work/fastp_reports
```

The folder roles are:

- `work/clean_reads/` — cleaned paired FASTQ files; and
- `work/fastp_reports/` — an HTML and JSON quality report for each sample.

These files can be regenerated from the preserved files in `raw_data/`.

## 3. Create the sample-name list

The cleaning and recovery steps need a consistent sample identifier. For the example data, the identifier is the part of each filename before `_R1` or `_R2`.

Create a file containing one sample name per line:

```bash
printf "%s\n" \
  GenusA_speciesA_CAP \
  GenusB_speciesB_CAP \
  GenusC_speciesC_CAP \
  GenusD_speciesD_CAP \
  > work/sample_names.txt
```

Display it:

```bash
cat work/sample_names.txt
```

The result should be:

```text
GenusA_speciesA_CAP
GenusB_speciesB_CAP
GenusC_speciesC_CAP
GenusD_speciesD_CAP
```

This small text file controls which samples are processed in Parts 2 and 3. The same approach will scale to the later *Echinocereus* dataset by replacing these four lines with its sample names.

Sample names should be unique and should not contain spaces.

## 4. Clean all paired reads with fastp

Run `fastp` once for each sample:

```bash
while read -r sample
do
  fastp \
    --in1 "raw_data/angiosperms353_example/${sample}_R1.fq.gz" \
    --in2 "raw_data/angiosperms353_example/${sample}_R2.fq.gz" \
    --out1 "work/clean_reads/${sample}_R1.clean.fq.gz" \
    --out2 "work/clean_reads/${sample}_R2.clean.fq.gz" \
    --detect_adapter_for_pe \
    --html "work/fastp_reports/${sample}.fastp.html" \
    --json "work/fastp_reports/${sample}.fastp.json" \
    --thread 4
done < work/sample_names.txt
```

The `while` loop reads one sample name at a time from `work/sample_names.txt`. For each sample:

- `--in1` and `--in2` identify its original R1 and R2 files;
- `--out1` and `--out2` write a new cleaned pair;
- `--detect_adapter_for_pe` enables additional paired-end adapter detection;
- `--html` creates a report that is easy to inspect in a browser;
- `--json` creates the same report in a machine-readable form; and
- `--thread 4` allows fastp to use four processor threads.

This tutorial uses fastp's default quality and length filters rather than introducing dataset-specific thresholds. It removes failing read pairs while keeping the R1 and R2 outputs synchronized.

The files in `raw_data/` are only read. All cleaned reads are written to `work/clean_reads/`.

## 5. Review a fastp report

List the cleaned read pairs:

```bash
ls -lh work/clean_reads
```

There should be eight cleaned FASTQ files: R1 and R2 for each of four samples.

For a detailed explanation of the report, read [Reading a fastp HTML Report](02a_reading_a_fastp_html_report.md). It uses `GenusB_speciesB_CAP` as a worked example and assumes no previous sequencing experience.

## 6. Create a 10-gene tutorial target

The downloaded Angiosperms353 target file contains 353 loci. HybPiper normally performs a separate assembly for every locus that has matching reads, which can take a long time on a laptop.

For this tutorial, extract the first 10 unique genes into a new target file:

```bash
awk '
  /^>/ {
    locus = $0
    sub(/^.*-/, "", locus)

    if (!(locus in selected) && number_selected < 10) {
      selected[locus] = 1
      number_selected++
    }

    keep_record = (locus in selected)
  }

  keep_record
' references/angiosperms353/Angiosperms353_targetSequences.fasta \
  > references/angiosperms353/Angiosperms353_first_10_genes.fasta
```

Check the number of unique loci in the smaller file:

```bash
grep "^>" references/angiosperms353/Angiosperms353_first_10_genes.fasta \
  | sed "s/.*-//" \
  | sort -u \
  | wc -l
```

The result should be:

```text
10
```

This subset is intended only for learning and checking the pipeline. The tree produced from it will be a tutorial result rather than a final analysis of all 353 loci.

## 7. End of Part 2

The new files should resemble:

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
│   └── fastp_reports/
│       └── four HTML and four JSON reports
└── tutorial/
    ├── 01_environment_setup_and_data_retrieval.md
    ├── 02_read_cleaning_and_target_preparation.md
    └── 02a_reading_a_fastp_html_report.md
```

Part 3 will use HybPiper to recover genes from the cleaned reads, summarize recovery, and collect one unaligned FASTA file per gene.
