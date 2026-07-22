# Part 5: Phylogenetic Tree Construction and Interpretation

## Inputs

- `work/concatenated/angiosperms353_tutorial.fasta`.
- `work/concatenated/angiosperms353_tutorial_partitions.txt`.

## Outputs

- The maximum-likelihood tree `results/iqtree/angiosperms353_tutorial.treefile`.
- The main report `results/iqtree/angiosperms353_tutorial.iqtree`.
- Bootstrap, model-selection, partition-scheme, checkpoint, and log files in `results/iqtree/`.

Part 5 will:

1. infer a maximum-likelihood tree with [IQ-TREE](https://iqtree.github.io/);
2. estimate statistical support for its internal branch; and
3. interpret the resulting tree without claiming more than the tutorial data can show.

The result is a preliminary, unrooted tree from four anonymized samples and nine recovered genes. It demonstrates the workflow but is not a final biological analysis.

## 1. Open the project and activate the environment

Open Terminal and enter:

```bash
cd "/Users/gauravmahajan/Documents/phylogeny"
conda activate angiosperms353
```

Confirm that IQ-TREE is available:

```bash
iqtree -version
```

Run all remaining commands from the `phylogeny/` project directory.

## 2. Confirm the Part 4 inputs

Display the sample names in the concatenated alignment:

```bash
grep "^>" work/concatenated/angiosperms353_tutorial.fasta
```

The result must be:

```text
>GenusA_speciesA_CAP
>GenusB_speciesB_CAP
>GenusC_speciesC_CAP
>GenusD_speciesD_CAP
```

Confirm the number of samples:

```bash
grep -c "^>" work/concatenated/angiosperms353_tutorial.fasta
```

The result must be `4`. If it is larger, do not construct the tree. Rerun Part 4 beginning with Section 4 so the HybPiper annotations are removed from the sample headers before concatenation.

Display the gene partitions:

```bash
cat work/concatenated/angiosperms353_tutorial_partitions.txt
```

There should be one line for each of the nine recovered genes. Each line gives the gene's start and end columns in the concatenated alignment.

## 3. Understand what IQ-TREE will estimate

IQ-TREE uses **maximum likelihood**. It compares possible tree topologies and evolutionary-model parameters, then finds the combination under which the observed alignment has the highest likelihood.

This does not mean that IQ-TREE calculates the probability that the chosen tree is true. It means that this tree explains the observed data better than the alternatives IQ-TREE examined under the selected models.

The partition file initially allows every gene to have its own model. IQ-TREE can merge partitions that are adequately described by the same model. Merging models does not mix up the gene boundaries; it reduces unnecessary model complexity.

The analysis will also calculate two support measures:

- **SH-aLRT support** tests whether an internal branch is better supported than nearby alternatives.
- **Ultrafast bootstrap support (UFBoot)** measures how consistently a branch is recovered after resampling alignment columns.

Both are branch-support measures, not guarantees that a biological relationship is correct.

## 4. Infer the tree

Create a directory for the IQ-TREE results:

```bash
mkdir -p results/iqtree
```

Run:

```bash
iqtree \
  -s work/concatenated/angiosperms353_tutorial.fasta \
  -p work/concatenated/angiosperms353_tutorial_partitions.txt \
  -st DNA \
  -m MFP+MERGE \
  -B 1000 \
  --alrt 1000 \
  -T AUTO \
  -seed 12345 \
  --prefix results/iqtree/angiosperms353_tutorial
```

The arguments mean:

- `-s` supplies the concatenated alignment;
- `-p` supplies the gene partitions;
- `-st DNA` explicitly identifies the sequences as DNA;
- `-m MFP+MERGE` selects substitution models and tests whether partitions can share a model;
- `-B 1000` performs 1,000 ultrafast-bootstrap replicates;
- `--alrt 1000` performs 1,000 SH-aLRT replicates;
- `-T AUTO` lets IQ-TREE choose the number of processor cores;
- `-seed 12345` makes the stochastic parts of the tutorial repeatable; and
- `--prefix` gives every result file a consistent location and name.

Warnings that some columns contain only gaps or ambiguous bases, or that a particular partition has no parsimony-informative sites, are expected for this small and partially recovered tutorial dataset. They do not mean the command failed. A message beginning with `ERROR` does.

IQ-TREE writes a checkpoint file while it runs. If the run is interrupted, repeating the same command allows it to resume.

## 5. Identify the important output files

List the results:

```bash
ls results/iqtree
```

The most useful files are:

- `angiosperms353_tutorial.treefile` — the maximum-likelihood tree with branch lengths and support values;
- `angiosperms353_tutorial.iqtree` — the main human-readable report;
- `angiosperms353_tutorial.contree` — the ultrafast-bootstrap consensus tree;
- `angiosperms353_tutorial.best_scheme.nex` — the selected models and merged partition scheme; and
- `angiosperms353_tutorial.log` — the messages printed during the run.

The `.treefile` is the main tree to retain and visualize. The `.iqtree` report explains how it was obtained.

## 6. Read the Newick tree

Display the tree:

```bash
cat results/iqtree/angiosperms353_tutorial.treefile
```

The tree is stored in **Newick format**. A shortened example looks like:

```text
(GenusA_speciesA_CAP,
 (GenusB_speciesB_CAP,GenusD_speciesD_CAP)100/100,
 GenusC_speciesC_CAP);
```

The symbols mean:

- sample names are the tree tips;
- parentheses group tips separated by the same internal branch;
- `100/100` gives SH-aLRT support followed by UFBoot support; and
- numbers following colons in the actual file are branch lengths.

With the supplied data and the fixed seed, the tutorial tree should support the unrooted split:

```text
GenusB_speciesB_CAP + GenusD_speciesD_CAP
versus
GenusA_speciesA_CAP + GenusC_speciesC_CAP
```

Small differences in branch-length decimals can occur with different IQ-TREE versions.

## 7. Interpret branch support and branch length

The support label `100/100` means that the split received:

- 100% SH-aLRT support; and
- 100% ultrafast-bootstrap support.

This is strong support within this alignment and analysis. It is not proof that the relationship is biologically correct. Incorrect sequence recovery, paralogs, alignment problems, model limitations, or disagreement among genes can still produce a highly supported concatenated tree.

A branch length is measured approximately in expected substitutions per aligned site. A longer branch indicates more inferred sequence change, not automatically more elapsed time. Estimating time requires additional molecular-clock assumptions and calibration information.

The tips represent the sequenced samples. If several individuals of one species are included in the later *Echinocereus* dataset, each individual will appear as its own tip.

## 8. Understand why the tree is unrooted

The substitution models used here estimate an **unrooted** tree. The placement at the left or base of a printed tree is only a drawing choice and does not identify the ancestor.

The example sample names are anonymized, so this tutorial does not choose an outgroup. For the real dataset, rooting should use a sample chosen from biological knowledge to lie outside the main group of interest.

IQ-TREE can write the output tree with a chosen outgroup by adding:

```text
-o OUTGROUP_SAMPLE_NAME
```

Do not select an outgroup merely because it appears most different. The decision should be made before looking at the result and supported by prior taxonomic or phylogenetic knowledge.

## 9. View the report

Open the main report in Terminal:

```bash
less results/iqtree/angiosperms353_tutorial.iqtree
```

Inside `less`, type `/MAXIMUM LIKELIHOOD TREE` and press Enter to find IQ-TREE's text drawing of the tree. Press `q` to exit.

For a graphical figure, the `.treefile` can also be opened in a phylogenetic tree viewer. Changing the rotation of branches in a viewer does not change the topology.

## 10. State the limits of the tutorial result

This tree uses:

- only four anonymized samples;
- nine recovered genes from a 10-gene teaching subset;
- direct nucleotide alignments;
- no detailed alignment filtering; and
- no biologically chosen outgroup.

It therefore shows that the commands can produce an interpretable tree, but it cannot validate the known relationships among the unnamed source species.

A full *Echinocereus* analysis should later use the broader locus set, examine paralog warnings and alignments, evaluate missing data, choose an appropriate outgroup, and examine disagreement among genes. Those decisions should be made before treating the result as a species-level evolutionary hypothesis.

## 11. End of Part 5

The final tutorial structure should resemble:

```text
phylogeny/
├── work/
│   └── concatenated/
│       ├── angiosperms353_tutorial.fasta
│       └── angiosperms353_tutorial_partitions.txt
├── results/
│   └── iqtree/
│       ├── angiosperms353_tutorial.treefile
│       ├── angiosperms353_tutorial.iqtree
│       ├── angiosperms353_tutorial.contree
│       └── other IQ-TREE result files
└── tutorial/
    └── 05_phylogenetic_tree_construction.md
```

You have now taken paired Angiosperms353 FASTQ reads through cleaning, HybPiper locus recovery, alignment, concatenation, and preliminary phylogenetic tree inference.
