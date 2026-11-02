# klebsiella_AMR_NF
Nextflow pipeline for Klebsiella pneumoniae genome typing and carbapenemase genotype-phenotype concordance
# klebsiella_AMR_NF

A Nextflow pipeline that types *Klebsiella pneumoniae* genome assemblies and checks how well carbapenemase gene detection agrees with lab-measured carbapenem susceptibility.

> **Status: in development (November 2026).** The pipeline and results are not finished yet. This file is updated as each part is completed. Sections marked *planned* describe work that is not yet in the repository.

## Question

In public *K. pneumoniae* genomes that have lab-measured carbapenem results, how well does a simple written rule (carbapenemase gene detected or not) agree with the lab result? What is observed in the isolates where the two disagree?

## Scope

This project measures agreement for one rule on one drug class. It is a research and learning pipeline. It is **not** a clinical tool and must not be used to guide treatment.

Not covered: other antibiotic classes, machine learning, core-genome phylogeny.

## Background

Carbapenems are among the last antibiotics available for serious Gram-negative infections. Carbapenem-resistant Enterobacterales, which include *K. pneumoniae*, are on the WHO priority pathogen list. Most carbapenem resistance in *K. pneumoniae* comes from carbapenemase enzymes (for example KPC, NDM and OXA-48-like). Some isolates are resistant without a carbapenemase, often through changes in the outer membrane porins OmpK35 and OmpK36 combined with other beta-lactamases. Because of this, carbapenemase detection is a screening rule, not a full prediction of resistance.

## Pipeline overview

| Step | Process | Runs per | What it does |
| --- | --- | --- | --- |
| 0 | CHECK_AMRFINDER_DB | Once | Confirms the AMRFinderPlus database exists and records its version |
| 1 | CHECK_SAMPLESHEET | Isolate | Checks input columns, file paths and unique sample IDs |
| 2 | QUAST | Isolate | Assembly quality: total length, contigs, N50, GC content |
| 3 | AMRFINDERPLUS | Isolate | Resistance genes and point mutations |
| 4 | MLST | Isolate | 7-gene sequence type |
| 5 | KLEBORATE | Isolate | Species confirmation, sequence type, resistance genes, porin calls |
| 6 | MASH_SKETCH | Isolate | Genome sketch for distance calculation |
| 7 | MASH_DIST | Cohort | Pairwise genome distances |
| 8 | SUMMARISE | Cohort | One row per isolate with QC flags, exclusion decisions and carbapenemase status |
| 9 | CONCORDANCE | Analysis | Joins lab phenotypes; 2 x 2 tables with 95% Wilson intervals |
| 10 | FIGURES | Analysis | Summary figures |

Lab phenotypes are joined only at step 9, so carbapenemase status is set without seeing them.

## Data

- **Source:** [NCBI Pathogen Detection](https://www.ncbi.nlm.nih.gov/pathogens/) (Isolates Browser and AST Browser).
- **Selection:** *K. pneumoniae* isolates with a public assembly and a meropenem susceptibility record. The exact query, download date and counts are recorded in `docs/cohort_counts.md` (*planned*).
- **Raw genomes are not stored in this repository.** The accession list and download commands are provided so the data can be downloaded again.

Lab results are split into two tiers that are analysed separately and never combined:

| Tier | Records | How they are labelled | Use |
| --- | --- | --- | --- |
| Tier 1 | Meropenem MIC with a documented method | Reinterpreted under one breakpoint standard (edition stated in `docs/phenotype_rules.md`) | Main analysis |
| Tier 2 | Submitter susceptible/intermediate/resistant call without a usable MIC | Original call and standard kept | Exploratory, reported separately |

Missing MICs are never filled in. Intermediate results are excluded from the main analysis and tested in a sensitivity analysis.

## Rules

### Carbapenemase status

| Value | Rule |
| --- | --- |
| `carbapenemase_detected` | AMRFinderPlus reports a complete gene whose subclass contains CARBAPENEM |
| `carbapenemase_uncertain` | The only carbapenemase hit is partial or at a contig end |
| `carbapenemase_not_detected` | No carbapenemase hit |

Full details: `docs/genotype_rules.md` (*planned*).

### Assembly QC

QC flags and exclusion decisions are stored in separate columns. Every isolate stays visible in the output table.

| Check | Source | Default | Effect |
| --- | --- | --- | --- |
| Species | Kleborate | *K. pneumoniae* | Exclude if different |
| Total length | QUAST | 5.0 to 6.5 Mb | Exclude if outside |
| Contigs | QUAST | 500 or fewer | Flag only |
| N50 | QUAST | 20 kb or more | Flag only |
| GC content | QUAST | about 56% to 58.5% | Flag only |

Thresholds are set as parameters in `nextflow.config` and can be changed without editing code.

## Requirements

- Linux, or Windows with WSL2 (developed on Ubuntu under WSL2)
- [Miniforge](https://github.com/conda-forge/miniforge) (conda and mamba)
- Tested with 4 CPU cores and 4 GB RAM available to WSL2

## Software versions

| Tool | Version | Purpose |
| --- | --- | --- |
| Nextflow | TBD | Workflow engine |
| OpenJDK | 17 | Required by Nextflow |
| NCBI Datasets CLI | TBD | Genome download |
| QUAST | TBD | Assembly QC |
| AMRFinderPlus | 4.2.7 | Resistance genes and mutations |
| AMRFinderPlus database | TBD | Reference gene catalogue |
| mlst | 2.35.0 | Sequence typing |
| Kleborate | 3.2.4 | Klebsiella typing |
| Mash | 2.3 | Genome distances |
| Python | 3.11 | Summary, statistics, figures |

A complete list with every Python package version will be in `results/software_versions.tsv` (*planned*).

## Installation

Current manual setup (each tool has its own conda environment):

```bash
conda config --add channels bioconda
conda config --add channels conda-forge
conda config --set channel_priority strict

mamba create -n nf -y openjdk=17 nextflow ncbi-datasets-cli python=3.11 pandas scipy statsmodels matplotlib biopython pytest
mamba create -n amrfinder -y ncbi-amrfinderplus
mamba create -n mlst -y mlst
mamba create -n kleborate -y kleborate
mamba create -n quast -y quast python=3.11
mamba create -n mash -y mash

mkdir -p ~/db/amrfinder
conda run -n amrfinder amrfinder_update -d ~/db/amrfinder
```

*Planned:* environment files with pinned versions in `envs/`, used by the pipeline directly.

## Usage (*planned*)

```bash
# Small test run (3 genomes)
nextflow run main.nf -profile test,conda --amrfinder_db ~/db/amrfinder/latest

# Full run
nextflow run main.nf -profile conda --input data/samplesheet.csv --amrfinder_db ~/db/amrfinder/latest
```

## Outputs (*planned*)

| File | Content |
| --- | --- |
| `isolate_summary.tsv` | One row per isolate: QC metrics, flags, exclusion decision, species, sequence type, carbapenemase status |
| `phenotype_curated.tsv` | Curated lab results: MIC, units, method, standard, tier, inclusion status |
| `concordance.tsv` | 2 x 2 counts, sensitivity, specificity, discordance rates and 95% intervals |
| `discordant_review.tsv` | Observed evidence for each discordant isolate, with an evidence level |
| `software_versions.tsv` | Tool and database versions |
| `run_manifest.tsv` | Input accessions and processing status |

## Results

Not yet available.

## Known limitations

- Public lab results come from many laboratories, methods and breakpoint versions.
- Short-read assemblies can split genes across contigs, which can make a gene look partial or missing.
- Public collections over-represent resistant and outbreak isolates, so the numbers describe this cohort and not a clinical population.
- Isolates from the same transmission cluster are not fully independent.
- Only one drug class (carbapenems) is analysed.

## Repository layout (*planned*)

```
main.nf              workflow
nextflow.config      parameters and profiles
modules/             one file per process
envs/                conda environment files
bin/                 Python scripts
assets/test_data/    small test genomes
tests/               unit tests
docs/                rules and cohort notes
```

## References

- AMRFinderPlus: Feldgarden M, et al. AMRFinderPlus and the Reference Gene Catalog facilitate examination of the genomic links among antimicrobial resistance, stress response, and virulence. *Scientific Reports* 11, 12728 (2021).
- Kleborate: Lam MMC, et al. A genomic surveillance framework and genotyping tool for *Klebsiella pneumoniae* and its related species complex. *Nature Communications* 12, 4188 (2021).
- mlst: Seemann T. https://github.com/tseemann/mlst. Uses PubMLST schemes (Jolley KA, et al. *Wellcome Open Research* 3, 124 (2018)).
- Mash: Ondov BD, et al. Mash: fast genome and metagenome distance estimation using MinHash. *Genome Biology* 17, 132 (2016).
- QUAST: Gurevich A, et al. QUAST: quality assessment tool for genome assemblies. *Bioinformatics* 29, 1072 to 1075 (2013).
- Nextflow: Di Tommaso P, et al. Nextflow enables reproducible computational workflows. *Nature Biotechnology* 35, 316 to 319 (2017).

## Author

Shnaya Mehla

## License

MIT. See `LICENSE`.

Tool	Version
Nextflow	26.10.4
NCBI Datasets	18.40.0
AMRFinderPlus	4.2.7
mlst	2.35.0
Kleborate	3.2.4
QUAST	5.3.0
Mash	2.3
