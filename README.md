[![English](https://img.shields.io/badge/Language-English-2563eb)](README.md)
[![中文](https://img.shields.io/badge/Language-中文-0f766e)](README_zh-CN.md)

# Flora

Flora is an end-to-end workflow for full-length single-cell RNA sequencing. It is optimized for MGI C4 single-cell libraries sequenced with the Cyclone long-read platform.

Flora performs:

- full-length cDNA identification, orientation, trimming, and chimeric-read rescue;
- dual-end barcode extraction, correction, merging, and cell assignment;
- UMI extraction and directional clustering;
- gene and isoform expression matrix generation;
- RNA QC and saturation analysis;
- Scanpy RNA clustering and UMAP visualization;
- generation of a self-contained HTML report.

[Glycine](https://github.com/CycloneSEQ-Bioinformatics/Glycine) is integrated into Flora. Users do not need to install Glycine separately or provide `--glycine-bin-dir`.

## Platform

The current binary release requires:

- Linux x86_64;
- glibc 2.35 or newer (the release is built on Ubuntu 22.04/glibc 2.35);
- Conda, Miniforge, Mambaforge, or Micromamba;
- Python 3.11, installed by the bundled environment file.

The release is not compatible with macOS, ARM Linux, or Windows.

Check the system architecture and glibc version before downloading:

```bash
uname -m
ldd --version | head -n 1
```

The expected architecture is `x86_64`. Older HPC operating systems, including systems with glibc 2.17, cannot run this binary release.

## Download

Download the latest archive from [GitHub Releases](https://github.com/brilliantlee2/Flora/releases).

For Flora v0.1.0:

```bash
wget https://github.com/brilliantlee2/Flora/releases/download/v0.1.0/Flora-0.1.0-linux-x86_64.tar.gz
wget https://github.com/brilliantlee2/Flora/releases/download/v0.1.0/Flora-0.1.0-linux-x86_64.tar.gz.sha256

sha256sum -c Flora-0.1.0-linux-x86_64.tar.gz.sha256
tar -xzf Flora-0.1.0-linux-x86_64.tar.gz
cd Flora-0.1.0-linux-x86_64
```

## Install the runtime environment

```bash
export LANG=C.UTF-8
export LC_ALL=C.UTF-8
export PYTHONUTF8=1

conda env create -f environment.yml
conda activate flora
```

Verify the installation:

```bash
python --version
samtools --version
minimap2 --version
bedtools --version

./flora --version
./flora --help
./flora mixed --help
./flora glycine --help
```

The embedded Python analysis modules require Python 3.11. Do not replace the environment with Python 3.10, 3.12, 3.13, or 3.14.

For reproducible runs, always provide the 10 bp barcode whitelist explicitly with `--barcode-list-10bp /path/to/BC_1536.txt`. Automatic discovery of a whitelist placed next to the executable is not guaranteed in the current release.

## Prepare a reference

`--ref-dir` must contain:

```text
reference/
├── genome.fa
├── genes.gtf
├── genes.bed
└── chrom_sizes.tsv
```

Generate the auxiliary files with:

```bash
samtools faidx genome.fa
paftools.js gff2bed -j genes.gtf > genes.bed
cut -f1,2 genome.fa.fai | sort -V > chrom_sizes.tsv
```

`paftools.js` is distributed with minimap2.

Chromosome or contig names must match across `genome.fa`, `genes.gtf`, `genes.bed`, and `chrom_sizes.tsv`. For example, do not mix `chr1` with `1` between files. An explicit `--isoform-gtf` can be supplied when the isoform annotation differs from `genes.gtf`; otherwise Flora reuses `genes.gtf`.

## Analyze raw FASTQ

The integrated Glycine stage runs automatically:

```bash
./flora \
  --fastq /data/sample.fastq.gz \
  --barcode-list-10bp /data/BC_1536.txt \
  --ref-dir /data/GRCh38_flora \
  --out-dir ./sample_output \
  --sample-id sample \
  --threads 32 \
  --cluster-threads 16 \
  --top1-alpha 0.1 \
  --max-ed 2
```

## Analyze an existing full-length FASTQ

```bash
./flora \
  --skip-glycine \
  --full-length-fastq /data/sample.full-length-plus-rescued.fq.gz \
  --barcode-list-10bp /data/BC_1536.txt \
  --ref-dir /data/GRCh38_flora \
  --out-dir ./sample_output \
  --sample-id sample \
  --threads 32 \
  --cluster-threads 16 \
  --top1-alpha 0.1 \
  --max-ed 2
```

## Mixed-species analysis

```bash
./flora mixed \
  --skip-glycine \
  --full-length-fastq /data/mixed.full-length-plus-rescued.fq.gz \
  --barcode-list-10bp /data/BC_1536.txt \
  --ref-dir /data/merged_reference \
  --out-dir ./mixed_output \
  --sample-id mixed_sample \
  --threads 32 \
  --cluster-threads 16 \
  --top1-alpha 0.1 \
  --max-ed 2
```

Mixed-species runs additionally generate `qc/barnyard_qc/barnyard_summary.tsv`, `barnyard_per_cell.tsv`, and a Barnyard QC section in the HTML report.

## Initial resource guidance

Resource use depends on read count, read length, reference size, barcode diversity, storage speed, and thread count. These are conservative starting points rather than guaranteed limits.

| Compressed FASTQ | Threads | Requested RAM | Initial wall-time allowance |
|---:|---:|---:|---:|
| up to 5 GB | 16-24 | 96-128 GB | 4-8 h |
| 5-20 GB | 24-32 | 192-256 GB | 12-24 h |
| 20-50 GB | 32 | 384-512 GB | 24-48 h |
| 50-100 GB | 32-48 | 768 GB-1 TB | 48-96 h |

Increasing the number of threads can increase peak memory and does not guarantee proportional acceleration. Benchmark a representative sample before reducing scheduler memory requests.

As an initial storage estimate, reserve at least 3-5 times the compressed FASTQ size as writable scratch space when using the default light-output mode. Full output and retained intermediates can require substantially more space. Monitor both space and inodes with `df -h` and `df -i` during the first production run.

## HPC scheduler example

The exact resource syntax depends on the cluster. The following Sun Grid Engine example requests 32 CPU slots and 256 GB RAM:

```bash
qsub -cwd \
  -l vf=256G,p=32 \
  -binding linear:32 \
  -P PROJECT_NAME \
  -q QUEUE_NAME \
  flora_job.sh
```

Inside `flora_job.sh`, activate the runtime environment and run `./flora` or `./flora mixed` in the foreground. Do not append `&` to the Flora command inside a submitted job script. Resource names such as `vf`, `p`, project, queue, and binding policy must be adapted to the local scheduler configuration.

## Outputs

The output directory contains:

```text
upstream/    Barcode correction, cell assignment, and knee plots
alignment/   Aligned and tagged BAM files
matrix/      Gene/isoform matrices and RNA clustering coordinates
qc/          RNA QC, saturation, and report inputs
logs/        Per-stage logs
```

Key outputs include `read_assigned_cell.csv`, `barcode_to_cell.csv`, tagged BAM files, gene and isoform expression matrices, Scanpy UMAP coordinates, and `<sample>.single_cell_report.html`. Mixed-species analysis also reports the per-cell human/mouse UMI classification under `qc/barnyard_qc/`.

## Support

Please report reproducible problems through [GitHub Issues](https://github.com/brilliantlee2/Flora/issues). Include the Flora version, command, operating system, input size, requested resources, and the relevant stage log. Do not upload private FASTQ/BAM data.

## License notices

No project-wide license has been declared for Flora. The integrated Glycine component retains its MIT license and attribution; third-party notices are included in every release archive.
