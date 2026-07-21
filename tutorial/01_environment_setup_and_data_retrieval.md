# Part 1: Environment Setup and Data Retrieval

This tutorial series will show how to turn Angiosperms353 sequencing reads into a preliminary phylogeny. It is divided into five parts:

1. **Environment setup and data retrieval** — this document
2. Read cleaning and tutorial target preparation
3. Angiosperms353 locus recovery with HybPiper
4. Locus alignment, filtering, and concatenation
5. Phylogenetic tree construction and interpretation

We will first work with a small example containing four plant samples. The data were generated using the Angiosperms353 probe set and come from the [Captus example-data tutorial](https://edgardomortiz.github.io/captus.docs/tutorials/assembly/basic/index.html). We are using only its example reads—not the Captus software.

Our main program will be [HybPiper](https://github.com/mossmatters/HybPiper). HybPiper recovers targeted genes from high-throughput sequencing reads. Later parts will use MAFFT to align the recovered genes and IQ-TREE to infer a phylogeny.

Part 1 does not process reads or construct a tree. It prepares the software and downloads the two required inputs:

1. paired-end Angiosperms353 sequencing reads; and
2. an Angiosperms353 target-sequence FASTA file.

## 1. Understand the two HybPiper inputs

### Sequencing reads: FASTQ

Raw Illumina data are usually delivered as compressed FASTQ files:

```text
sample1_R1.fastq.gz
sample1_R2.fastq.gz
```

FASTQ stores both the called DNA bases and a quality score for every base. `R1` and `R2` are reads from opposite ends of the same DNA fragments, so they must remain correctly paired.

### Target sequences: FASTA

HybPiper also needs a FASTA file containing known sequences for the targeted genes. It uses these sequences to recognize which reads belong to each Angiosperms353 locus.

The official nucleotide target file is named:

```text
Angiosperms353_targetSequences.fasta
```

It contains 4,781 representative sequences covering 353 loci. Multiple representative sequences are provided for many loci because Angiosperms353 is intended to work across flowering plants.

## 2. Open the renamed project directory

Open Terminal and enter:

```bash
cd "/Users/gauravmahajan/Documents/phylogeny"
```

Confirm the location:

```bash
pwd
```

The result should be:

```text
/Users/gauravmahajan/Documents/phylogeny
```

Run the remaining commands from this directory unless a later step says otherwise.

## 3. Install Conda

Conda will manage HybPiper and its supporting programs in one isolated environment. We will install Miniforge, a small Conda distribution maintained by conda-forge.

First check whether Conda is already installed:

```bash
conda --version
```

If this prints a version number, skip to Section 4.

If Terminal reports `command not found`, confirm that the Mac uses Apple Silicon:

```bash
uname -m
```

The expected result is `arm64`.

Download the current Apple Silicon Miniforge installer from the official [Miniforge repository](https://github.com/conda-forge/miniforge):

```bash
curl -fsSLo Miniforge3.sh "https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-MacOSX-arm64.sh"
```

Run the installer:

```bash
bash Miniforge3.sh
```

During installation:

1. review the license and enter `yes` to accept it;
2. press Enter to accept the default installation location; and
3. enter `yes` when asked whether to update the shell profile and automatically initialize Conda.

Answering `yes` to the final question allows Terminal to find the `conda` command. Although the question mentions activation at startup, we will disable automatic activation of the `base` environment below.

Remove the installer after setup:

```bash
rm Miniforge3.sh
```

Close and reopen Terminal, then confirm the installation:

```bash
conda --version
```

Prevent the `base` environment from activating automatically whenever Terminal opens:

```bash
conda config --set auto_activate_base false
```

Conda will remain available, but the command prompt will change only after you deliberately activate an environment.

If you answered `no` to shell initialization and `conda` is not found, initialize it manually:

```bash
~/miniforge3/bin/conda init zsh
source ~/.zshrc
conda config --set auto_activate_base false
conda --version
```

HybPiper itself runs on Apple Silicon, but its complete Conda environment includes programs distributed for Intel Macs. Install Rosetta 2 so macOS can run them:

```bash
softwareupdate --install-rosetta --agree-to-license
```

If Rosetta is already installed, macOS will say so and no change is needed.

## 4. Create the HybPiper environment

Create an Intel-compatible environment named `angiosperms353`:

```bash
CONDA_SUBDIR=osx-64 conda create -n angiosperms353 \
  -c conda-forge -c bioconda \
  hybpiper=2.3.4 fastp iqtree
```

Enter `y` when Conda asks for confirmation.

This installs:

- HybPiper and its required assembly/recovery programs;
- `fastp`, which Part 2 will use to clean the FASTQ reads; and
- IQ-TREE, which Part 5 will use to estimate a phylogeny.

Activate the environment:

```bash
conda activate angiosperms353
```

Keep future package installations in this environment on the same Intel-compatible platform:

```bash
conda config --env --set subdir osx-64
```

Confirm that HybPiper is available:

```bash
hybpiper --help
```

Check that HybPiper can find its supporting programs:

```bash
hybpiper check_dependencies
```

Also confirm the read-cleaning and tree programs:

```bash
fastp --version
iqtree -version
```

Some IQ-TREE installations name the executable `iqtree2`. If `iqtree` is not found, use:

```bash
iqtree2 -version
```

Keep this environment active while following the tutorial. It can later be deactivated with `conda deactivate`.

## 5. Create folders for immutable inputs

Create separate locations for original sequencing reads and reference sequences:

```bash
mkdir -p raw_data/angiosperms353_example
mkdir -p references/angiosperms353
```

The distinction is:

- `raw_data/` contains original sequencing deliveries; and
- `references/` contains the target sequences used to identify loci.

Neither directory should receive cleaned reads, assemblies, alignments, or trees. Those will go into working and results directories in later parts.

## 6. Download and extract the example Angiosperms353 reads

Download the four-sample example archive:

```bash
curl -L \
  "https://drive.usercontent.google.com/download?export=download&id=1Jq3raXEBP8D_X9yEWOh9FF3YTSq6vZAT&confirm=t" \
  -o raw_data/angiosperms353_example/00_raw_reads.tar.gz
```

Extract its FASTQ files directly into `raw_data/angiosperms353_example/`:

```bash
tar -xzf raw_data/angiosperms353_example/00_raw_reads.tar.gz \
  -C raw_data/angiosperms353_example \
  --strip-components=1
```

The archive is needed only for extraction, so delete it:

```bash
rm raw_data/angiosperms353_example/00_raw_reads.tar.gz
```

The download is approximately 178 MB. It contains one R1/R2 FASTQ pair for each of four anonymized plant samples:

```text
GenusA_speciesA_CAP
GenusB_speciesB_CAP
GenusC_speciesC_CAP
GenusD_speciesD_CAP
```

## 7. Download the Angiosperms353 target FASTA

Download the nucleotide target sequences from the official [Angiosperms353 repository](https://github.com/mossmatters/Angiosperms353):

```bash
curl -L \
  "https://raw.githubusercontent.com/mossmatters/Angiosperms353/f0698e2d0821bdc0214d1a022cf85454b3be24dd/Angiosperms353_targetSequences.fasta" \
  -o references/angiosperms353/Angiosperms353_targetSequences.fasta
```

The long identifier in the URL is a Git commit. Using it instead of `master` ensures that the tutorial continues to retrieve the same target-file version if the repository changes later.

This is a nucleotide target file. In Part 3, HybPiper will therefore receive it through the nucleotide-target option and will use BWA for read mapping.

## 8. Confirm and protect the downloaded inputs

List the extracted FASTQ files:

```bash
ls -lh raw_data/angiosperms353_example/*.fq.gz
```

You should see eight files: an R1 and R2 file for each of the four samples.

Count the target FASTA records:

```bash
grep -c "^>" references/angiosperms353/Angiosperms353_targetSequences.fasta
```

The target file should contain `4781` FASTA records. This is larger than 353 because many loci have representative sequences from more than one flowering plant.

## 9. Why we are not using HybPiper’s own test reads

HybPiper provides an excellent official test dataset with nine samples and 13 target genes. However, those genes come from an *Artocarpus* bait set, not from Angiosperms353.

This tutorial instead combines:

- a small example containing genuine Angiosperms353 reads; and
- the official Angiosperms353 nucleotide targets;

then processes them with HybPiper. This more closely matches the intended *Echinocereus* analysis.

## 10. End of Part 1

The project should now have this structure:

```text
phylogeny/
├── raw_data/
│   └── angiosperms353_example/
│       ├── GenusA_speciesA_CAP_R1.fq.gz
│       ├── GenusA_speciesA_CAP_R2.fq.gz
│       ├── GenusB_speciesB_CAP_R1.fq.gz
│       ├── GenusB_speciesB_CAP_R2.fq.gz
│       ├── GenusC_speciesC_CAP_R1.fq.gz
│       ├── GenusC_speciesC_CAP_R2.fq.gz
│       ├── GenusD_speciesD_CAP_R1.fq.gz
│       └── GenusD_speciesD_CAP_R2.fq.gz
├── references/
│   └── angiosperms353/
│       └── Angiosperms353_targetSequences.fasta
└── tutorial/
    └── 01_environment_setup_and_data_retrieval.md
```

The `angiosperms353` Conda environment contains HybPiper, fastp, and IQ-TREE. The extracted example reads and target FASTA are preserved separately and have not been processed.

Part 2 will clean the FASTQ files with fastp and create a smaller target for the tutorial. Part 3 will use HybPiper to recover target loci from each sample.
