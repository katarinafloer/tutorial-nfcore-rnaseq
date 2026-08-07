# RNA-seq nf-core Unity Run Notes

This repository is for running and testing nf-core RNA-seq workflows on Unity
HPC. It contains scripts, notes, and troubleshooting details from setting up
`nf-core/demo` and `nf-core/rnaseq` with Apptainer, Slurm, scratch workspaces,
and a custom *Staurois parvus* EGAPx genome.

This is not the polished tutorial website. It is the working/run repository.

## Current Scripts

The Unity scripts live in:

```bash
/home/kfloer_smith_edu/rnaseq_nf_core/scripts
```

Current scripts:

```text
demo_nfcore.sh
nfcore-rnaseq-test.sh
test-nfcore-tadpole-genome.sh
```

The useful pattern is:

```bash
sbatch /home/kfloer_smith_edu/rnaseq_nf_core/scripts/<script-name>.sh
```

The Slurm log files are written to:

```bash
/home/kfloer_smith_edu/rnaseq_nf_core/job-logs
```

## Tested Pipeline Stages

We tested the setup in stages:

1. `nf-core/demo` with `-profile test,unity`.
2. `nf-core/rnaseq` with `-profile test,unity`.
3. `nf-core/rnaseq` with a custom EGAPx *Staurois parvus* genome and a small
   subset of real paired-end FASTQ files.

The successful approach was to run the Nextflow controller through `sbatch` on a
compute node instead of launching directly from the login node.

## Unity Modules

The working versions were:

```bash
module purge
module load nextflow/26.04.1
module load apptainer/latest
```

Nextflow may print that a newer version is available. That is just an update
notice unless the run fails because of a version requirement.

## Scratch Workspaces

Use Unity scratch workspaces for active runs:

```bash
ws_allocate rnaseq_real_test 30
ws_list
```

For the custom-genome test, the scratch workspace was:

```bash
/scratch/workspace/kfloer_smith_edu-rnaseq_real_test
```

The custom genome files were:

```bash
GENOME_FASTA=/scratch3/workspace/kfloer_smith_edu-simple/egapx_sparvus/output/complete.genomic.fna
GENOME_GTF=/scratch3/workspace/kfloer_smith_edu-simple/egapx_sparvus/output/complete.genomic.gtf
```

The test FASTQs were:

```text
C45-1B_S2_L002
/work/pi_lmangiamele_smith_edu/03_26_flut_yale_rnaseq/C45-1B_S2_L002_R1_001.fastq.gz
/work/pi_lmangiamele_smith_edu/03_26_flut_yale_rnaseq/C45-1B_S2_L002_R2_001.fastq.gz

C45-1H_S1_L002
/work/pi_lmangiamele_smith_edu/03_26_flut_yale_rnaseq/C45-1H_S1_L002_R1_001.fastq.gz
/work/pi_lmangiamele_smith_edu/03_26_flut_yale_rnaseq/C45-1H_S1_L002_R2_001.fastq.gz
```

## Cache Layout

Inside each scratch workspace, the scripts create:

```text
.apptainer/build-cache/        Apptainer OCI/blob cache
.apptainer/tmp/                Temporary directory for Apptainer and proot
.nextflow-apptainer-cache/     Final Nextflow container .img files
work/                          Nextflow task work directories
results_*/                     Pipeline outputs
samplesheets/                  Generated samplesheets
```

These paths are set explicitly so container downloads and temporary files do not
fill home or use fragile default temp directories.

## Important Environment Variables

The scripts should set:

```bash
export APPTAINER_CACHEDIR="$SCRATCH_RNASEQ/.apptainer/build-cache"
export APPTAINER_TMPDIR="$SCRATCH_RNASEQ/.apptainer/tmp"
export PROOT_TMP_DIR="$SCRATCH_RNASEQ/.apptainer/tmp"
export NXF_APPTAINER_CACHEDIR="$SCRATCH_RNASEQ/.nextflow-apptainer-cache"
export NXF_OPTS='-Xms1g -Xmx4g'
export PROOT_NO_SECCOMP=1
```

Why they matter:

- `APPTAINER_CACHEDIR`: stores container layers while Apptainer builds images.
- `APPTAINER_TMPDIR`: gives Apptainer a scratch temp directory.
- `PROOT_TMP_DIR`: prevents proot from using `/tmp`, which may be mounted
  `noexec`.
- `NXF_APPTAINER_CACHEDIR`: stores final `.img` files for Nextflow reuse.
- `NXF_OPTS`: keeps the Nextflow Java controller small.
- `PROOT_NO_SECCOMP=1`: works around proot/kernel restrictions during image
  builds.

## Troubleshooting Notes From This Run

### Boolean Parameters Were Parsed as Strings

Passing boolean parameters directly on the command line caused nf-core schema
validation errors:

```text
Value is [string] but should be [boolean]
```

The robust fix was to write a JSON params file:

```json
{
  "igenomes_ignore": true,
  "aligner": "star_salmon",
  "pseudo_aligner": "salmon",
  "skip_bbsplit": true,
  "skip_markduplicates": true,
  "skip_rseqc": true,
  "skip_dupradar": true
}
```

Then launch with:

```bash
nextflow run nf-core/rnaseq \
  -r 3.26.0 \
  -profile unity \
  -params-file "$SCRATCH_RNASEQ/tadpole_params.json" \
  -resume
```

### Corrupt Apptainer Images

Several failed container builds left behind bad `.img` files. The error looked
like:

```text
not a valid squashfs image
```

The scripts clean these before launching:

```bash
find "$NXF_APPTAINER_CACHEDIR" -type f -name "*.pulling.*" -delete || true

find "$NXF_APPTAINER_CACHEDIR" -type f -name "*.img" | while read -r img; do
    if ! apptainer inspect "$img" > /dev/null 2>&1; then
        echo "Removing invalid container image: $img"
        rm -f "$img"
    fi
done
```

This removes incomplete or invalid containers, but keeps completed Nextflow task
outputs.

### Registry or Network Errors

Some pulls failed with registry-side errors such as:

```text
502 Bad Gateway
```

For these, resubmit the same script. `-resume` will reuse completed work once the
registry responds normally.

### `/tmp` Was Mounted `noexec`

One Apptainer/proot failure said:

```text
the current temporary directory (/tmp) is mounted with no execution permission
Please set PROOT_TMP_DIR env. variable to an alternate location
```

The fix is:

```bash
export PROOT_TMP_DIR="$SCRATCH_RNASEQ/.apptainer/tmp"
```

This should be included in all Unity run scripts.

### What `-resume` Does

`-resume` does not fix errors by itself. It prevents completed tasks from being
rerun after the underlying problem is fixed.

Useful pattern:

1. Read the error.
2. Fix the cause, such as a corrupt container image or bad parameter.
3. Resubmit the same script with `-resume`.

## Monitoring Jobs

Check your jobs:

```bash
squeue --me
```

Estimate start times:

```bash
squeue --me --start
```

Inspect a job:

```bash
scontrol show job <JOBID>
```

Follow logs:

```bash
tail -f /home/kfloer_smith_edu/rnaseq_nf_core/job-logs/<log-file>.out
tail -f /home/kfloer_smith_edu/rnaseq_nf_core/job-logs/<log-file>.err
```

## Practical Run Order

For a fresh setup, use this order:

1. Run `demo_nfcore.sh`.
2. Run `nfcore-rnaseq-test.sh`.
3. Run `test-nfcore-tadpole-genome.sh` on the small real-data subset.
4. Scale up to all samples only after the custom-genome subset succeeds.

This keeps failures easier to diagnose.
