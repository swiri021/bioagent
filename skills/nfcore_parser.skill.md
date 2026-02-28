# nf-core Pipeline Parser SKILL

## Role
Convert any nf-core pipeline into a pipeline SKILL.md that BioAgent can execute.

## How to Access nf-core Code
1. Clone or browse the nf-core pipeline repository:
   ```
   git clone https://github.com/nf-core/{pipeline_name}.git
   ```
2. Or browse directly via GitHub API / raw content

## What to Read

### 1. Main Workflow (priority: HIGH)
- `workflows/{pipeline_name}.nf` or `main.nf`
- Extract: step order, channel connections (= dependencies), conditional logic

### 2. Modules (priority: HIGH)
For each process/module used in the workflow:
- `modules/nf-core/{tool_name}/main.nf`
- Extract:
  - `container` directive → docker image URI
  - `script` block → actual command to run
  - `input` / `output` declarations → file patterns
  - `cpus`, `memory`, `time` → resource requirements

### 3. Config (priority: MEDIUM)
- `nextflow.config` → default parameters
- `conf/base.config` → default resource allocations per process label
  - Example: `withLabel: 'process_high'` → cpus=12, memory=72GB
- `conf/modules.config` → per-module argument overrides

### 4. Input Schema (priority: MEDIUM)
- `assets/schema_input.json` or `nextflow_schema.json`
- Extract: required input format (samplesheet columns, file types)

### 5. MultiQC Config (priority: LOW)
- `assets/multiqc_config.yml`
- Extract: which QC metrics are collected

## Output Format

Generate a pipeline SKILL.md with this structure:

```markdown
# {pipeline_name}.skill.md

## Description
{Brief description from nf-core README}

## Input
- Format: {samplesheet format or direct file input}
- Required files: {e.g., paired-end FASTQ, genome FASTA, index}
- Example: {minimal example}

## Reference Data
- {List any required reference files: genome index, GTF, etc.}
- {Include download commands or known locations}

## Steps

### step_1: {tool_name}
- image: {docker image URI from container directive}
- cmd: {command from script block, parameterized}
- input: {input file patterns}
- output: {output file patterns}
- resources:
  - cpu: {from config label or process definition}
  - memory: {from config label or process definition}
  - disk: {estimate based on data type}
- depends_on: [] (none for first step)

### step_2: {tool_name}
- image: {docker image URI}
- cmd: {command}
- input: {input file patterns — reference outputs from previous steps}
- output: {output file patterns}
- resources: {cpu, memory, disk}
- depends_on: [step_1]

{... continue for all steps}

## QC Criteria
- {tool}: {metric} {threshold} (extract from pipeline documentation or common standards)

## Common Failure Modes
- {Known issues from nf-core GitHub issues or documentation}
```

## Rules
- Extract ONLY the essential execution logic, not Nextflow-specific syntax
- Always resolve container URIs to docker-compatible format (not singularity)
- If a module has `if/else` logic, document both branches
- Parameterize commands with `{variable}` placeholders
- Include ALL steps, even optional ones (mark them as optional)
- If resource requirements aren't explicit, use these defaults:
  - process_low: cpu=2, memory=4GB
  - process_medium: cpu=4, memory=8GB
  - process_high: cpu=8, memory=32GB
  - process_high_memory: cpu=8, memory=72GB
