# PubMed Gene Analysis Pipeline

A Nextflow pipeline that retrieves and analyzes PubMed publication counts for human protein-coding genes. This pipeline demonstrates parallelized processing across thousands of genes using Nextflow's workflow orchestration.

## Overview

This pipeline scrapes PubMed to count publications associated with each human protein-coding gene, providing insights into research activity across the genome.

## Pipeline Architecture

The pipeline consists of three sequential processes:

![](pubmed.png)

### 1. `prepare`
Downloads and processes the NCBI Homo sapiens gene information database:
- Fetches gene data from NCBI FTP server
- Filters for protein-coding genes only
- Removes duplicate gene symbols
- Outputs:
  - `genes.csv`: List of unique gene IDs for processing
  - `genes_hash_table.csv`: Mapping of gene IDs to symbols

### 2. `analyze` (Parallelized)
Web scrapes PubMed for each gene ID in parallel:
- Queries PubMed database for human-specific publications (1900-2030)
- Extracts publication counts from search results
- Configured with `maxForks 5` to respect NCBI rate limits
- Outputs: Individual CSV files per gene with publication counts
- Error handling: Creates error logs for failed requests

### 3. `summarize`
Aggregates results across all genes:
- Combines individual gene results
- Maps gene IDs to gene symbols
- Sorts by gene symbol
- Outputs: `summary.csv` with gene symbols and publication counts

## Quick Start

### Prerequisites
- [Nextflow](https://www.nextflow.io/) (≥21.04)
- Docker or container runtime
- AWS credentials (for S3 output, optional)

### Local Execution

```bash
# Clone the repository
git clone https://github.com/alethiotx/pubmed.git
cd pubmed

# Run with Docker (local profile)
nextflow run main.nf -profile local

# Test mode (processes only 20 genes)
nextflow run main.nf -profile local --env test
```

### Production Execution

```bash
# Run with S3 output and full gene set
nextflow run main.nf -profile seqera --outdir s3://your-bucket/path/
```

## Configuration

### Profiles

- **`local`**: Docker-based execution with local output directory
- **`seqera`**: Cloud execution with container orchestration

### Parameters

- `params.outdir`: Output directory path (default: S3 bucket in nextflow.config)
- `params.env`: Environment mode (`prod` or `test`). Test mode processes only 20 genes.

### Rate Limiting

The NCBI E-utilities allow:
- 3 requests/second without API key
- 10 requests/second with API key

The pipeline uses `maxForks 5` in the `analyze` process to stay within rate limits and avoid blocking.

## Container Image

Public ECR image: `public.ecr.aws/alethiotx/pubmed:latest`

Built from Ubuntu 25.10 with:
- Python 3 virtual environment
- Required packages: biopython, pandas, beautifulsoup4, urllib3

## CI/CD

GitHub Actions workflow automatically builds and pushes Docker images to Amazon ECR Public on every push. See `.github/workflows/docker-deploy.yaml` for details.

## Infrastructure

Terraform configuration in `terraform/` manages:
- ECR Public repository
- Public image pull policy
- IAM permissions for GitHub Actions OIDC

## Output Structure

```
output/
├── prepare/
│   ├── genes.csv
│   └── genes_hash_table.csv
└── summarize/
    └── summary.csv
```

## Development

### Python Scripts

- `bin/prepare.py`: Gene list preparation from NCBI data
- `bin/analyze.py`: PubMed scraping for individual genes
- `bin/summarize.py`: Result aggregation and formatting

### Testing

Run in test mode to validate changes:
```bash
nextflow run main.nf -profile local --env test
```

## License

See [LICENSE](LICENSE) for details.

## Acknowledgments

Data sources:
- NCBI Gene database
- PubMed/NCBI E-utilities