# SparvusGenome

Scripts and notes for assembling, QC-ing, submitting, and annotating the *Staurois parvus* genome.

This repository documents the workflow used to generate a new PacBio HiFi genome assembly for the Bornean rock frog / foot-flagging frog, *Staurois parvus*, screen it for NCBI submission, and annotate it with NCBI EGAPx.

## Project overview

The main goals of this workflow are:

1. Generate a new *S. parvus* genome assembly from PacBio HiFi reads.
2. Assess assembly quality using summary statistics, BUSCO, GC/coverage checks, and contamination screening.
3. Clean the assembly for NCBI submission.
4. Annotate the cleaned genome using NCBI EGAPx with *S. parvus* RNA-seq evidence.
5. Prepare outputs for downstream RNA-seq mapping and differential gene expression analyses.

## Current assembly summary

The cleaned primary-contig assembly used for NCBI submission and EGAPx annotation has:

| Metric | Value |
|---|---:|
| Total assembly length | 4,481,589,302 bp / 4.48 Gb |
| Number of contigs | 4,081 |
| Largest contig | 36.84 Mb |
| Contig N50 | 6.88 Mb |
| N50 number | 182 |
| N90 | 629 kb |
| Gaps / Ns | 0 |
| GC content | 43.07% |

BUSCO, using `vertebrata_odb10`:

| BUSCO category | Count | Percent |
|---|---:|---:|
| Complete BUSCOs | 3,060 | 91.2% |
| Complete single-copy | 2,952 | 88.0% |
| Complete duplicated | 108 | 3.2% |
| Fragmented | 134 | 4.0% |
| Missing | 160 | 4.8% |
| Total searched | 3,354 | 100% |

BUSCO notation:

```text
C:91.2%[S:88.0%,D:3.2%],F:4.0%,M:4.8%,n:3354
```

## Repository structure

This repository currently contains scripts for several stages of the project.

### Assembly

| Script/file | Purpose |
|---|---|
| `sparvus_hifiasm.sh` | Run hifiasm assembly workflow. |
| `sparvus_config.sh` | Shared project paths/configuration variables for assembly scripts. |
| `final_stats.sh` | Generate final assembly statistics. |
| `GenomeAssemblyInteractiveDump.ipynb` | Interactive notebook used during assembly exploration/QC. |

### Assembly QC

| Script/file | Purpose |
|---|---|
| `2_sparvus_busco.sh` | Run BUSCO on the assembly. |
| `run_gc_coverage_check.sh` | Run GC/coverage screening to identify potential contaminants or outliers. |
| `run_gc_coverage_subset.sh` | Run GC/coverage checks on a subset. |

### NCBI contamination/adaptor screening

| Script/file | Purpose |
|---|---|
| `download_fcs_gx_db.sh` | Download the NCBI FCS-GX database. |
| `run_fcs_adaptor.sh` | Run NCBI FCS adaptor screening. |
| `run_fcs_gx.sh` | Run NCBI FCS-GX contamination screening. |
| `remove_contaminant.sh` | Remove the contaminant contig identified by FCS-GX. |

The final cleaned assembly removed one contaminant contig:

```text
Contig: ptg003819l
Length: 19,494 bp
Assignment: nematode / Oscheius tipulae
Action: EXCLUDE
```

This represented approximately 0.0004% of the assembly.

### EGAPx annotation

| Script/file | Purpose |
|---|---|
| `validate_inputs_annot.sh` | Validate EGAPx input files before annotation. |
| `run_sparvus_egapx.sbatch` | Initial EGAPx SLURM run script. |
| `resume_sparvus_egapx.sbatch` | Resume EGAPx after a failed/interrupted run. |
| `egapx_config/` | Snapshot of EGAPx config files edited for Unity/SLURM. |
| `egapx_instructions.md` | Detailed step-by-step EGAPx setup and run notes. |

The initial EGAPx run failed because `run_mask_assm_stats` ran out of memory. The run was successfully resumed after increasing memory for that process.

EGAPx version used:

```text
EGAPx 0.5.2
EGAPx commit: 40ec7362576e4b93fa9c5bf0a6c400d9502b8a63
Support data version: egapxsupportdata_20251017
```

EGAPx annotation summary:

| Feature category | Count |
|---|---:|
| Total genes | 41,598 |
| Protein-coding genes | 23,550 |
| mRNAs | 30,699 |
| CDSs | 30,699 |
| Non-coding RNAs | 14,814 |
| Pseudo transcripts | 3,691 |

In the GFF, the EGAPx total gene count corresponds to:

```text
37,929 gene features
+ 3,669 pseudogene features
= 41,598 total gene/pseudogene features
```

Isoforms are not counted as separate genes in the GFF. They are represented as `mRNA` child features under parent `gene` features:

```text
32,415 mRNA records
25,246 parent genes with at least one mRNA
4,231 genes with more than one mRNA isoform
```

### Adult RNA-seq download for annotation rerun

| Script/file | Purpose |
|---|---|
| `download_GSE232203_adult_rnaseq.sbatch` | Download adult male *S. parvus* RNA-seq data from SRA/GEO. |
| `compress_GSE232203_existing_fastq.sbatch` | Compress partially converted FASTQ files after `fasterq-dump`. |

Adult RNA-seq source:

```text
GEO: GSE232203
BioProject: PRJNA971214
Species: Staurois parvus
Samples: adult male brain, spinal cord, and leg muscle
```

These data may be added to the RNA-seq evidence for a future EGAPx rerun.

### Conda environments

| File | Purpose |
|---|---|
| `assembly_env.yml` | Assembly-related tools. |
| `busco_env.yml` | BUSCO environment. |
| `annot_env.yml` | Annotation-related tools. |
| `detonate_env.yml` | DETONATE/transcriptome QC environment. |
| `sp_ont_polish.yml` | ONT polishing environment. |

## EGAPx input structure

EGAPx was run using:

- cleaned genome assembly
- NCBI taxid for *Staurois parvus*: `386267`
- paired Illumina RNA-seq reads
- tRNAscan enabled
- cmsearch enabled

Example EGAPx YAML:

```yaml
genome: /scratch3/workspace/kfloer_smith_edu-simple/sparvus_hifiasm/assembly_output/sparvus_primary_contigs.ncbi.fa
taxid: 386267
short_reads: /scratch3/workspace/kfloer_smith_edu-simple/egapx_sparvus/input/short_reads.txt

trnascan:
  enabled: true

cmsearch:
  enabled: true
```

The short-read manifest is a two-column file:

```text
sample_name /path/to/R1.fastq.gz
sample_name /path/to/R2.fastq.gz
```

See `egapx_instructions.md` for the full setup and run details.

## Androgen receptor sanity check

Because androgen receptor is a gene of particular interest for this project, the EGAPx annotation was checked for `AR`.

EGAPx recovered one androgen receptor locus:

| AR feature | EGAPx result |
|---|---|
| Contig | `ptg000154l` |
| Coordinates | 1,491,873-1,803,334 |
| Strand | `+` |
| Gene ID | `gene-egapxtmp_016766` |
| Locus tag | `egapxtmp_016766` |
| Transcript ID | `rna-egapxtmp_016766-R1` |
| Protein ID | `egapxtmp_016766-P1` |
| Exons | 8 |
| Predicted protein length | 777 aa |
| Ortholog cross-reference | `NCBIOrtholog:367` |

BLASTP of the predicted protein against anuran proteins returned top hits to androgen receptor proteins. The top hit was:

```text
androgen receptor [Aquarana catesbeiana]
Query cover: 100%
E-value: 0.0
Percent identity: 92.86%
Hit length: 777 aa
```

This supports the EGAPx AR model as a likely near-complete/full-length androgen receptor ortholog.

## Notes for reuse

Most scripts are specific to the Unity HPC environment and will need editing before use elsewhere.

Likely things to modify:

- SLURM partition/account/qos settings
- module names
- file paths
- Apptainer/Singularity cache and tmp directories
- memory/CPU/time values
- Nextflow configuration
- EGAPx config files

The scripts should be treated as reproducible project documentation and templates, not universal drop-in workflows.

## Suggested workflow order

A rough order for using this repository is:

1. Configure project paths in `sparvus_config.sh`.
2. Run assembly with `sparvus_hifiasm.sh`.
3. Generate assembly statistics with `final_stats.sh`.
4. Run BUSCO with `2_sparvus_busco.sh`.
5. Run GC/coverage checks.
6. Run NCBI FCS adaptor and FCS-GX contamination screening.
7. Remove contaminant contig if needed.
8. Validate cleaned assembly.
9. Prepare EGAPx input YAML and RNA-seq manifest.
10. Validate EGAPx inputs with `validate_inputs_annot.sh`.
11. Run EGAPx with `run_sparvus_egapx.sbatch`.
12. If needed, adjust EGAPx process resources and resume with `resume_sparvus_egapx.sbatch`.
13. Use EGAPx GFF/GTF/protein/transcript outputs for downstream RNA-seq mapping and DGE.

