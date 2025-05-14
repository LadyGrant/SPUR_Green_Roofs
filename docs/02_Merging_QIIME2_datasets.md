# QIIME2: Merging Multi-Year Amplicon Datasets

This workflow merges amplicon sequencing data from two separate QIIME2 runs (e.g., 2023 and 2024) **after denoising (DADA2)** but **before taxonomy classification**.

Why Merge Before Taxonomy?    
QIIME2 assigns ASV IDs based on exact sequence. Merging before taxonomy ensures that:
- Shared sequences across years get the same ASV ID
- You only need to assign taxonomy once
- All downstream diversity and taxa plots reflect consistent biology


## Prerequisites

Before merging:
- Both years must be trimmed to **the same lengths** (e.g., `--p-trunc-len-f 220`, `--p-trunc-len-r 200`)
- Both years must be run through **DADA2 independently**
- You should have:
  - `table_2023.qza`
  - `rep-seqs_2023.qza`
  - `table_2024.qza`
  - `rep-seqs_2024.qza`

⚠️ **Important: Avoid Duplicate Sample IDs Across Runs**

QIIME2 does not allow merging datasets that contain duplicate sample IDs. If you're working with multi-year or multi-batch data, make sure **all sample names are unique** before processing.

Recommended: Append the year or batch to all sample IDs (e.g., `H3RZ_A_2023`, `NTC1_2024`)

If duplicates are present, QIIME2 will raise an error during merging, and you’ll need to reprocess at least up to the DADA2 step with corrected sample names.


## Step 1: Merge Feature Tables

```
source /home/opt/Miniconda3/miniconda3/bin/activate qiime2-2023.9

qiime feature-table merge \
  --i-tables table_2023.qza \
  --i-tables table_2024.qza \
  --o-merged-table table_merged.qza
```
## Step 2: Merge Representative Sequences
```
qiime feature-table merge-seqs \
  --i-data rep-seqs_2023.qza \
  --i-data rep-seqs_2024.qza \
  --o-merged-data rep-seqs_merged.qza
```

Next Steps    

After merging:    
- Assign taxonomy to rep-seqs_merged.qza    
- Filter contaminants (e.g., chloroplasts, mitochondria, Eukaryota)    
- Proceed with downstream analysis





