# 🧬 BioAgent

**Bioinformatics pipeline orchestration powered by Claude Code.**

Replace Nextflow/Snakemake with natural language. BioAgent reads nf-core pipeline logic and executes it on any infrastructure — no pipeline code to write or maintain.

## How It Works

```
You:    "I have 10 paired-end fastq files in gs://my-bucket/fastq/. 
         Run RNA-seq analysis using GCP Batch."

BioAgent: 1. Reads nf-core/rnaseq logic → builds execution plan
          2. Submits jobs to GCP Batch with correct containers
          3. Monitors progress, handles failures intelligently
          4. Delivers results + QC report
```

## What You Need

1. **Claude Code** (the orchestrator)
2. **This repo** (SKILL files + config)
3. **Cloud credentials** (GCP/AWS) or **Docker** (local)
4. **Your data**

That's it. No Groovy, no DSL, no pipeline code.

## Setup

```bash
# 1. Clone this repo
git clone https://github.com/your-org/bioagent.git
cd bioagent

# 2. Copy and edit config
cp bioagent.yaml.template bioagent.yaml
# Edit with your project IDs, buckets, credentials

# 3. Run with Claude Code
claude "Read the bioagent skills and config. 
       I have paired-end fastq files at gs://my-bucket/samples/.
       Run nf-core rnaseq pipeline on GCP Batch."
```

## Project Structure

```
bioagent/
├── bioagent.yaml              # Your configuration
├── skills/
│   ├── bioagent_core.skill.md # Main orchestration logic
│   ├── nfcore_parser.skill.md # nf-core → SKILL converter
│   ├── gcp_batch.skill.md     # GCP Batch executor
│   ├── aws_batch.skill.md     # AWS Batch executor
│   └── local_docker.skill.md  # Local Docker executor
└── README.md
```

## Supported Executors

| Executor | Use Case |
|----------|----------|
| **Local Docker** | Testing, small datasets, development |
| **GCP Batch** | Production, scalable, pay-per-use |
| **AWS Batch** | Production, scalable, pay-per-use |

## Supported Pipelines

Any nf-core pipeline. BioAgent reads the nf-core source code and extracts the execution logic automatically. Examples:

- **nf-core/rnaseq** — RNA sequencing analysis
- **nf-core/sarek** — WGS/WES variant calling
- **nf-core/atacseq** — ATAC-seq peak calling
- **nf-core/chipseq** — ChIP-seq analysis
- **nf-core/methylseq** — Bisulfite sequencing
- ... and 100+ more from the nf-core community

## Why Not Nextflow?

| | Nextflow | BioAgent |
|---|---------|----------|
| **Add a new pipeline** | Write Groovy DSL | "Run chipseq analysis" |
| **Change a tool** | Edit module, test, PR | "Use minimap2 instead of BWA" |
| **Switch cloud** | Rewrite config profiles | "Run this on AWS instead" |
| **Debug failure** | Dig through work/ dirs | BioAgent reads logs and explains |
| **Update pipeline** | Pull, resolve conflicts | Always reads latest nf-core code |
| **Custom analysis** | Write new pipeline | Describe in natural language |

## Example Usage

```bash
# RNA-seq on GCP
claude "Run rnaseq on my 10 fastq pairs in gs://bucket/data/ using GCP Batch.
        Use GRCh38 reference. Include salmon quantification."

# WGS variant calling locally
claude "I have WGS fastq files in /data/samples/. 
        Run sarek pipeline locally with Docker.
        Only do variant calling, skip annotation."

# Switch executor mid-project
claude "The GCP run is too expensive. 
        Re-run the failed samples on AWS Batch instead."

# Custom analysis
claude "Run fastqc on all samples first. 
        Show me the results before continuing.
        Then only proceed with samples that pass QC."
```

## License

MIT
