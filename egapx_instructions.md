# EGAPx instructions for *Staurois parvus*

These notes document the EGAPx annotation workflow used for the new *Staurois parvus* genome assembly. They are meant as a practical template for rerunning EGAPx on Unity or adapting the run to another HPC.

Main run scripts:

- Initial EGAPx run:  
  https://github.com/katarinafloer/SparvusGenome/blob/main/run_sparvus_egapx.sbatch

- Resume script after the first run hit an out-of-memory error:  
  https://github.com/katarinafloer/SparvusGenome/blob/main/resume_sparvus_egapx.sbatch

- EGAPx config snapshot:  
  https://github.com/katarinafloer/SparvusGenome/tree/main/egapx_config

Important note: the two SLURM scripts are not the entire setup. EGAPx also needs an input YAML, an RNA-seq manifest, the EGAPx repo/venv, Nextflow + Apptainer/Singularity, and cluster-specific config edits.

## 1. Project directory

For this run, I used:

```bash
PROJECT=/scratch3/workspace/kfloer_smith_edu-simple/egapx_sparvus
```

Create the project directory structure:

```bash
mkdir -p "$PROJECT"/{input,work,output,container_cache,tmp,logs}
```

## 2. Clone and install EGAPx

```bash
cd "$PROJECT"

git clone https://github.com/ncbi/egapx.git
cd "$PROJECT/egapx"

python3 -m venv venv
source venv/bin/activate

python -m pip install --upgrade pip
python -m pip install -r requirements.txt

python ui/egapx.py -V
git rev-parse HEAD | tee "$PROJECT/egapx_commit.txt"
```

For my run:

```text
EGAPx 0.5.2
commit: 40ec7362576e4b93fa9c5bf0a6c400d9502b8a63
```

## 3. Software/modules used on Unity

On Unity, I used:

```bash
module purge
module load apptainer/latest
module load nextflow/24.04.3
```

Python 3.12 was available on the system even though `module load python/3.12` did not work.

Check versions:

```bash
python3 --version
java -version
nextflow -version
apptainer --version
command -v singularity || command -v apptainer
```

## 4. EGAPx input YAML

The input YAML was:

```yaml
genome: /scratch3/workspace/kfloer_smith_edu-simple/sparvus_hifiasm/assembly_output/sparvus_primary_contigs.ncbi.fa
taxid: 386267
short_reads: /scratch3/workspace/kfloer_smith_edu-simple/egapx_sparvus/input/short_reads.txt

trnascan:
  enabled: true

cmsearch:
  enabled: true
```

Save as:

```text
/scratch3/workspace/kfloer_smith_edu-simple/egapx_sparvus/input/sparvus.yaml
```

Validate YAML syntax:

```bash
cd "$PROJECT/egapx"
source venv/bin/activate

python - <<'PY'
import yaml

path = "/scratch3/workspace/kfloer_smith_edu-simple/egapx_sparvus/input/sparvus.yaml"

with open(path) as handle:
    config = yaml.safe_load(handle)

print(config)
print("YAML syntax passed")
PY
```

## 5. RNA-seq manifest

EGAPx expects the short-read manifest as a two-column whitespace-delimited file:

```text
sample_name /path/to/read1.fastq.gz
sample_name /path/to/read2.fastq.gz
```

Example:

```text
C103-2B_S10_L002 /work/pi_lmangiamele_smith_edu/03_26_flut_yale_rnaseq/C103-2B_S10_L002_R1_001.fastq.gz
C103-2B_S10_L002 /work/pi_lmangiamele_smith_edu/03_26_flut_yale_rnaseq/C103-2B_S10_L002_R2_001.fastq.gz
```

For the initial run, I used 24 paired Illumina RNA-seq libraries from:

```bash
READDIR=/work/pi_lmangiamele_smith_edu/03_26_flut_yale_rnaseq
```

Create manifest:

```bash
PROJECT=/scratch3/workspace/kfloer_smith_edu-simple/egapx_sparvus
READDIR=/work/pi_lmangiamele_smith_edu/03_26_flut_yale_rnaseq
READLIST=$PROJECT/input/short_reads.txt
TMP_READLIST=$PROJECT/input/short_reads.txt.tmp

while IFS= read -r -d '' r1; do
    r2="${r1/_R1_001.fastq.gz/_R2_001.fastq.gz}"

    if [[ ! -r "$r2" ]]; then
        echo "ERROR: missing R2 for $r1" >&2
        exit 1
    fi

    filename=$(basename "$r1")
    setname=${filename%_R1_001.fastq.gz}

    printf '%s %s\n' "$setname" "$r1"
    printf '%s %s\n' "$setname" "$r2"
done > "$TMP_READLIST" < <(
    find "$READDIR" -maxdepth 1 -type f \
        -name '*_R1_001.fastq.gz' -print0 | sort -z
)

mv "$TMP_READLIST" "$READLIST"
```

Check manifest:

```bash
wc -l "$READLIST"
head -n 8 "$READLIST"

awk '
{ count[$1]++ }
END {
    for (sample in count)
        if (count[sample] != 2)
            print "ERROR:", sample, count[sample]
    print "Libraries:", length(count)
}
' "$READLIST"
```

For my initial run:

```text
48 lines
24 libraries
```

## 6. Validate input files

Optional but recommended: validate FASTQ gzip files and assembly basics before submitting EGAPx.

The validation script I used is here:

```text
/home/kfloer_smith_edu/assembly/scripts/validate_sparvus_egapx.sbatch
```

Validation output showed:

```text
All FASTQ gzip files passed.
Contigs: 4081
Total bases: 4481589302
Duplicate sequence IDs: 0
All input validation checks passed.
```

## 7. Generate EGAPx config files

The first EGAPx command generates config files and asks you to edit them.

```bash
PROJECT=/scratch3/workspace/kfloer_smith_edu-simple/egapx_sparvus
INPUT_YAML=$PROJECT/input/sparvus.yaml

module load apptainer/latest
module load nextflow/24.04.3

export APPTAINER_CACHEDIR="$PROJECT/container_cache"
export APPTAINER_TMPDIR="$PROJECT/tmp"
export SINGULARITY_CACHEDIR="$PROJECT/container_cache"
export SINGULARITY_TMPDIR="$PROJECT/tmp"
export NXF_SINGULARITY_CACHEDIR="$PROJECT/container_cache"
export TMPDIR="$PROJECT/tmp"

cd "$PROJECT/egapx"

python ui/egapx.py \
    "$INPUT_YAML" \
    -e slurm \
    -w "$PROJECT/work" \
    -o "$PROJECT/output"
```

EGAPx printed:

```text
For replicating this run use --data-version egapxsupportdata_20251017
Edit config files in .../egapx_config to reflect your actual configuration, then repeat the command
```

For reproducibility, I pinned:

```text
egapxsupportdata_20251017
```

## 8. Edit EGAPx config files

The active runtime config directory was:

```bash
CONFIG=/scratch3/workspace/kfloer_smith_edu-simple/egapx_sparvus/egapx/egapx_config
```

Files edited:

```text
slurm.config
process_resources.config
```

Important Unity-specific edits included:

- SLURM queue/partition: `cpu`
- Apptainer/Singularity cache directory in scratch
- Apptainer/Singularity tmp directory in scratch
- Process memory/CPU/time values

Useful checks:

```bash
grep -nE 'cacheDir|CACHEDIR|TMPDIR|queue =' "$CONFIG/slurm.config"
grep -nE 'params\.|memory =|cpus =|time =' "$CONFIG/process_resources.config"
```

My config snapshot is committed here:

```text
https://github.com/katarinafloer/SparvusGenome/tree/main/egapx_config
```

Important: these config files are cluster-specific. They will likely need to be edited for another HPC.

## 9. Initial EGAPx run script

The initial SLURM run script is:

```text
/home/kfloer_smith_edu/assembly/scripts/run_sparvus_egapx.sbatch
```

GitHub:

```text
https://github.com/katarinafloer/SparvusGenome/blob/main/run_sparvus_egapx.sbatch
```

The command used inside the script was essentially:

```bash
python ui/egapx.py \
    "$INPUT_YAML" \
    --data-version egapxsupportdata_20251017 \
    -e slurm \
    -w "$PROJECT/work" \
    -o "$PROJECT/output"
```

The driver SLURM job does not do most of the compute itself. Nextflow launches many child SLURM jobs.

## 10. OOM failure and fix

The first run failed at:

```text
egapx:winmask_plane:mask_assm_stats:run_mask_assm_stats
```

Error:

```text
exit status 137
Killed
```

Interpretation: out of memory.

Nextflow reported:

```text
To resume execution, run:
sh /scratch3/workspace/kfloer_smith_edu-simple/egapx_sparvus/output/nextflow/resume.sh
```

I added this process-specific override to `process_resources.config`:

```groovy
process {
    withName: 'run_mask_assm_stats' {
        memory = 256.GB
        time = 12.h
    }
}
```

Then I resumed.

## 11. Resume script

The resume SLURM script is:

```text
/home/kfloer_smith_edu/assembly/scripts/resume_sparvus_egapx.sbatch
```

GitHub:

```text
https://github.com/katarinafloer/SparvusGenome/blob/main/resume_sparvus_egapx.sbatch
```

It runs:

```bash
bash "$PROJECT/output/nextflow/resume.sh"
```

After resume, EGAPx completed successfully.

Final runtime summary:

```text
Completed at: 25-Jun-2026 04:27:48
Duration: 10h 33m 22s
CPU hours: 1,081.1
Succeeded: 804
Cached: 56
```

## 12. Main EGAPx outputs

Final outputs were in:

```text
/scratch3/workspace/kfloer_smith_edu-simple/egapx_sparvus/output
```

Main files:

```text
complete.genomic.gff
complete.genomic.gtf
complete.proteins.faa
complete.transcripts.fna
complete.cds.fna
complete.genomic.fna
```

Stats:

```text
output/stats/feature_counts.txt
output/stats/feature_counts.xml
output/stats/feature_stats.xml
output/stats/mask_assm_stats.xml
```

Find output files:

```bash
PROJECT=/scratch3/workspace/kfloer_smith_edu-simple/egapx_sparvus

find "$PROJECT/output" -type f \( \
    -name '*.gff' -o \
    -name '*.gff3' -o \
    -name '*.gtf' -o \
    -name '*.faa' -o \
    -name '*.fna' -o \
    -name '*.gbff' -o \
    -name '*.sqn' -o \
    -name '*.stats' -o \
    -name '*.xml' \
\) | sort
```

## 13. Annotation summary

From EGAPx:

```text
Total genes: 41,598
Protein-coding genes: 23,550
mRNAs: 30,699
CDSs: 30,699
Non-coding RNAs: 14,814
Pseudo transcripts: 3,691
```

In the GFF:

```text
37,929 gene features
3,669 pseudogene features
41,598 total gene + pseudogene features
```

Feature types in GFF:

```text
353957 exon
315309 CDS
37929 gene
32415 mRNA
8580 tRNA
3669 pseudogene
3493 snRNA
1822 rRNA
1739 lncRNA
470 snoRNA
384 pseudogenic_rRNA
282 transcript
19 scaRNA
10 V_gene_segment
8 C_gene_segment
```

## 14. Isoform check

EGAPx does not appear to classify isoforms as separate genes in the GFF.

Isoforms are represented as `mRNA` child features under parent `gene` features.

Command:

```bash
GFF=/scratch3/workspace/kfloer_smith_edu-simple/egapx_sparvus/output/complete.genomic.gff

awk -F'\t' '
$3=="mRNA" {
    parent="missing_parent"
    if (match($9, /Parent=[^;]+/)) {
        parent=substr($9, RSTART+7, RLENGTH-7)
    }
    mrna_count[parent]++
}
END {
    isoform_genes=0
    total_mrna=0
    for (g in mrna_count) {
        total_mrna += mrna_count[g]
        if (mrna_count[g] > 1) isoform_genes++
    }
    print "Genes with at least one mRNA:", length(mrna_count)
    print "Total mRNAs:", total_mrna
    print "Genes with >1 mRNA isoform:", isoform_genes
}
' "$GFF"
```

Output:

```text
Genes with at least one mRNA: 25246
Total mRNAs: 32415
Genes with >1 mRNA isoform: 4231
```

## 15. Gene naming / annotation usefulness

Direct GFF counts:

```text
All GFF gene features: 37,929
With gene symbol: 20,939
With NCBI ortholog cross-reference: 12,479
With description: 32,972
Without description: 4,957
```

Protein-coding genes:

```text
Total protein-coding genes: 23,550
With formal gene symbol: 12,359
With NCBI ortholog cross-reference: 12,479
With product/description text: 20,127
Without product/description: 3,423
```

## 16. Androgen receptor sanity check

EGAPx recovered one androgen receptor locus.

```text
Contig: ptg000154l
Coordinates: 1,491,873-1,803,334
Strand: +
Gene ID: gene-egapxtmp_016766
Locus tag: egapxtmp_016766
Transcript ID: rna-egapxtmp_016766-R1
Protein ID: egapxtmp_016766-P1
Exons: 8
Predicted protein length: 777 aa
Ortholog cross-reference: NCBIOrtholog:367
```

BLASTP against anuran proteins:

```text
Top hit: androgen receptor [Aquarana catesbeiana]
Query cover: 100%
E-value: 0.0
Percent identity: 92.86%
Hit length: 777 aa
```

This supports the EGAPx AR model as a likely near-complete/full-length AR ortholog.

## 17. Things to adapt on another HPC

The most HPC-specific pieces are:

- module names for Python, Nextflow, Apptainer/Singularity
- SLURM partition/queue name
- Apptainer/Singularity cache and tmp paths
- process memory/CPU/time values in `process_resources.config`
- whether the cluster uses Singularity or Apptainer
- whether Nextflow is installed as a module
- filesystem paths and bind paths

The provided scripts should be treated as templates, not drop-in scripts for every HPC.

