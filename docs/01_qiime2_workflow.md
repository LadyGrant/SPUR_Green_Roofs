# SPUR rooftop agrivoltaic 16S QIIME2 pipeline
## 1. Import Sequence Files into QIIME2

We used the **515F/806R V4 primers** from the Earth Microbiome Project (EMP): 
 [Earth Microbiome Project protocol](https://earthmicrobiome.org/protocols-and-standards/)

```
source /home/opt/Miniconda3/miniconda3/bin/activate qiime2-2023.9

qiime tools import \
  --type EMPPairedEndSequences \
  --input-path reads \
  --output-path emp-paired-end-sequences.qza
```
Explanation of Flags

| Flag            | Description                                                                                                         |
| --------------- | ------------------------------------------------------------------------------------------------------------------- |
| `--type`        | Specifies EMP format for paired-end reads (expects `forward.fastq.gz`, `reverse.fastq.gz`, and `barcodes.fastq.gz`) |
| `--input-path`  | Path to the folder containing the three raw FASTQ files                                                             |
| `--output-path` | Name of the output `.qza` file that wraps your imported reads in QIIME2's artifact format                           |

**Outputs**
`emp-paired-end-sequences.qza` QIIME2 artifact containing raw multiplexed paired-end sequences in EMP format 

## 2. Demultiplex the Sequences

```
#!/bin/bash
#SBATCH --nodes=1
#SBATCH --ntasks=8
#SBATCH --time=4-00:00:00
#SBATCH --mail-type=BEGIN,END,FAIL
#SBATCH --mem=128gb
#SBATCH --nodelist=zenith
#SBATCH --mail-user=First.Last@colostate.edu
#SBATCH --mail-type=BEGIN,END,FAIL
#SBATCH --partition=wrighton-hi,wrighton-low
#SBATCH --job-name=demux

cd ..

source /home/opt/Miniconda3/miniconda3/bin/activate qiime2-2023.9

qiime demux emp-paired \
  --m-barcodes-file metadata.txt \
  --m-barcodes-column barcode-sequence \
  --p-no-rev-comp-mapping-barcodes \
  --p-no-golay-error-correction \
  --i-seqs emp-paired-end-sequences.qza \
  --o-per-sample-sequences demux.qza \
  --o-error-correction-details demux-details.qza
```
Explanation of Flags

| Flag                                | Description                                                                                      |
|-------------------------------------|--------------------------------------------------------------------------------------------------|
| `--m-barcodes-file`                | Metadata file containing the barcode sequences for each sample                                  |
| `--m-barcodes-column`              | Column name in the metadata file that contains the barcode sequences                            |
| `--p-no-rev-comp-mapping-barcodes` | Prevents QIIME2 from reverse complementing the barcodes in the metadata file                    |
| `--p-no-golay-error-correction`    | Disables Golay error correction on barcode sequences (use when not using Golay barcodes)        |
| `--i-seqs`                         | The input artifact containing raw multiplexed EMP paired-end sequences                          |
| `--o-per-sample-sequences`         | Output artifact: demultiplexed paired-end sequences separated by sample                         |
| `--o-error-correction-details`     | Output file that includes stats on barcode matching and error correction (for QC/debugging)     |

**Notes:**
- `--p-no-rev-comp-mapping-barcodes`: Use this flag if your sequencing core confirmed that the barcodes are already in the correct orientation. If your barcodes were sequenced on the forward read (as is standard for 16S EMP protocols), this flag is typically appropriate.
- `--p-no-golay-error-correction`: ⚠️ Although our primers use 12-bp Golay barcodes (per the original EMP protocol), modern Illumina sequencing pipelines output exact barcode reads without applying Golay error correction. Therefore, we disable error correction using `--p-no-golay-error-correction` to ensure correct sample assignment during demultiplexing. This practice reflects updates in sequencing accuracy and lab demux workflows not reflected in older EMP documentation.

**Outputs**
`demux.qza` Per-sample demultiplexed paired-end sequences (used for DADA2)  
`demux-details.qza` Barcode matching and assignment statistics (useful for QC/troubleshooting)

## 3. Visualize sequence reads and sequence quality
After demultiplexing, it's important to assess the number of reads per sample and the quality of those reads before proceeding with denoising.
```
qiime demux summarize --i-data demux.qza --o-visualization demux.qzv
```
**Outputs** 
`demux.qzv` Interactive visualization of sequence counts and quality scores per sample. 

Open the resulting .qzv file in [QIIME2 View](https://view.qiime2.org/) to explore the interactive summary. 

## Overview of demux.qzv Summary Tabs

Demultiplexed Sequence Counts Summary
- Table with min, max, mean, median, and total read counts across all samples
- Helps you get a quick snapshot of sequencing depth
<img width="1072" alt="Screenshot 2025-05-08 at 7 49 45 PM" src="https://github.com/user-attachments/assets/a127a6a7-7a08-4481-a66b-2341c897c261" />

Forward and Reverse Reads Frequency Histograms
Show the distribution of sequence counts per sample for each read direction
Look for:
- A single main peak = consistent sequencing
- Long tails = outliers or uneven library concentration
<img width="1375" alt="Screenshot 2025-05-08 at 7 50 42 PM" src="https://github.com/user-attachments/assets/4b20fcc8-1f83-4272-b946-50cb38ae4a5c" />

Per-Sample Sequence Counts
Make sure samples expected to have high biomass have relatively high counts, and blanks/NTCs remain low. If anything looks suspicious, you may want to double-check your barcode assignments or rerun demux with updated metadata.

You can download the sample counts as a `.tsv` file for further inspection. If you're working from a plate map, it's a good idea to paste these read counts back into the plate layout. This helps you visually identify any systematic issues. For example, if an entire row or column has lower read counts, it could indicate a pipetting error, primer failure, or reagent issue during library prep or PCR.

Interactive Quality Plot
- Displays per-base quality scores for forward and reverse reads
- Use this plot to guide trimming settings for DADA2 (--p-trunc-len-f and --p-trunc-len-r)
- Look for median quality scores above Q30 in your usable regions

**Why are reverse reads usually lower in quality?**  
Reverse reads often show a faster drop-off in quality due to how Illumina sequencing-by-synthesis chemistry works. The longer the read, the more cycles the machine runs, which leads to increased error accumulation, especially toward the tail of the reverse read.

**How much can we trim while still keeping usable data?**  
You need to retain enough read length to ensure overlap for successful merging. For the 515F–806R primers targeting the V4 region of 16S rRNA:
- The expected amplicon length is ~253 bp
- Merging requires at least 20–30 bp of overlap between forward and reverse reads
- So, your trimmed forward + reverse read lengths should add up to at least **~273 bp**

Example:  
- Forward read trimmed to 220 bp  
- Reverse read trimmed to 200 bp  
- Total = 420 bp  
- Overlap = 420 – 253 = **167 bp**, plenty for successful merging

Reminder: Over-trimming will result in too little overlap for read merging; under-trimming can retain low-quality bases that increase error rates. Always use the quality plot to balance both.

⚠️ Although the 2023 reads have higher quality scores than 2024, we trimmed both datasets to the same length to ensure consistent ASV boundaries for merging.
<img width="1425" alt="Screenshot 2025-05-08 at 7 57 44 PM" src="https://github.com/user-attachments/assets/3544c0d1-c4be-4ab9-9299-ad871019eb39" />

## 4. Run DADA2 to denoise and merge sequence reads

```
#!/bin/bash
#SBATCH --nodes=1
#SBATCH --ntasks=4
#SBATCH --time=4-00:00:00
#SBATCH --mail-type=BEGIN,END,FAIL
#SBATCH --mem=20gb
#SBATCH --nodelist=zenith
#SBATCH --mail-user=First.Last@colostate.edu
#SBATCH --mail-type=BEGIN,END,FAIL
#SBATCH --partition=wrighton-hi,wrighton-low
#SBATCH --job-name=DADA2

cd ..

source /home/opt/Miniconda3/miniconda3/bin/activate qiime2-2023.9

qiime dada2 denoise-paired \
  --i-demultiplexed-seqs demux.qza \
  --p-trim-left-f 0 \
  --p-trim-left-r 0 \
  --p-trunc-len-f 220 \
  --p-trunc-len-r 200 \
  --o-table table.qza \
  --o-representative-sequences rep-seqs.qza \
  --o-denoising-stats denoising-stats.qza
```
### Explanation of Flags for `qiime dada2 denoise-paired`

| Flag                           | Description                                                                                  |
|--------------------------------|----------------------------------------------------------------------------------------------|
| `--i-demultiplexed-seqs`       | The input `.qza` file containing demultiplexed paired-end reads (from Step 2)                |
| `--p-trunc-len-f`              | Truncate forward reads at position 220. Removes low-quality tails.                           |
| `--p-trunc-len-r`              | Truncate reverse reads at position 200. Removes low-quality tails.                           |
| `--p-trim-left-f`              | Trims bases from the start of each forward read (used to remove primers if necessary)        |
| `--p-trim-left-r`              | Trims bases from the start of each reverse read (used to remove primers if necessary)        |
| `--o-table`                    | Output file containing the ASV feature table (samples x ASVs)                                |
| `--o-representative-sequences` | Output file containing representative sequences for each ASV                                 |
| `--o-denoising-stats`          | Output file containing read filtering and chimera removal statistics per sample              |
| `--p-n-threads`                | Number of CPU threads to use. Set to match available resources for faster processing         |

**Outputs**
`table.qza` Feature table with ASV counts per sample 
`rep-seqs.qza` FASTA-formatted sequences for each ASV  
`denoising-stats.qza` Summary of reads kept/lost at each DADA2 filtering step 

## 5. Visualize DADA2 outputs

Visualize denoising stats
```
qiime metadata tabulate \
--m-input-file denoising-stats.qza \
--o-visualization denoising-stats.qzv
```
### DADA2 Denoising Stats Explained

| Column                   | What it means                                             | What to look for                             |
|--------------------------|-----------------------------------------------------------|----------------------------------------------|
| `sample-id`              | Your sample names                                         | Check that all expected samples are present  |
| `input`                  | Raw paired-end read count per sample before any filtering | Samples <10,000 reads might be too shallow   |
| `filtered`               | Reads retained after quality filtering `--p-trunc-len)`   | Typically ~80–95% of input is good           |
| `percentage of input passed filter`| % of reads that passed filtering step           | <70% may suggest poor quality or trimming problems|
| `denoised`   | Reads that passed filtering and were successfully denoised (ASVs inferred) | Should be similar to filtered           |
| `merged`     | Successfully merged forward and reverse reads | This number drops if reads are too short or don’t overlap well       |
| `non-chimeric` | Merged reads after chimera removal    | This is what gets used in downstream analysis **(your final usable data)** |
| `percentage of input non-chimeric` | % of original reads that made it all the way through | <50% suggests high loss; ideally you want >60% of input|

Visualize feature-table
```
qiime feature-table summarize \
 --i-table table.qza  \
 --o-visualization table.qzv \
 --m-sample-metadata-file metadata.txt
```
## DADA2 Feature Table Summary Explained
This file summarizes your final ASV (amplicon sequence variant) table after DADA2. It contains **three pages** in QIIME2 View.

## Overall summary of samples and features - Page 1 

<img width="738" alt="Screenshot 2025-05-13 at 11 05 54 AM" src="https://github.com/user-attachments/assets/0058eb68-1097-429f-a4dd-e22fd221b749" />  


| Metric                            | Meaning                                                                                                        |
| --------------------------------- | ---------------------------------------------------------------------------------------------------------------|
| Number of samples      | Number of samples that passed **all steps** and are represented in your ASV table.  |
| Number of features     | This is your total count of **unique ASVs**. Each one represents a distinct sequence variant inferred by DADA2.|
| Total frequency        | This is the **sum of all read counts across all samples and all ASVs**. It’s your full sequencing depth after filtering, denoising, and chimera removal. |

<img width="1441" alt="Screenshot 2025-05-13 at 11 17 10 AM" src="https://github.com/user-attachments/assets/5ea8614c-3f20-4a94-a089-3c5b8c39e58c" />  
This section of the first page shows the distribution of read depth per sample. It's a quick way to assess how evenly your sequencing was distributed. Histogram on the right: Visually shows the distribution of sequencing depth across all your samples  

| Metric                              | What It Tells You                            |
| ----------------------------------- | -------------------------------------------- |
| Minimum frequency         | Your monst shallow sequenced sample, typically an NTC  |
| 1st quartile       | 25% of samples have fewer than this many reads                |
| Median frequency   | Half of your samples have fewer than \~69k reads              |
| 3rd quartile      | 75% of samples are under this value                            |
| Maximum frequenc  | Your most deeply sequenced sample                              |
| Mean frequency      | Average sequencing depth across all samples                  |
 

<img width="1421" alt="Screenshot 2025-05-13 at 11 33 25 AM" src="https://github.com/user-attachments/assets/3de99d99-fba7-41b9-8760-9997d0f8b483" />

This shows how abundant each ASV (feature) is across your dataset. It’s a key view for deciding how to filter low-abundance or rare features. Histogram on the right: Log-scale histogram of how many ASVs fall into different abundance ranges. Most ASVs are ultra-rare, with only a handful showing up more than a few hundred times. This is typical for microbiome datasets, especially soil and rhizosphere systems with long tails of diversity. 

| Metric                            | What It Tells You                                                                                  |
| --------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Minimum frequency       | Features (ASVs) that appear **only twice total** in your entire dataset — likly noise or extraction kit contaminants. |
| 1st quartile            | 25% of your ASVs have **6 reads or fewer** — likely your NTC's.                                                      |
| Median frequency       | Half of your ASVs have **fewer than 14 total reads** — this is very common in high-diversity systems like soil.    |
| 3rd quartile     | 75% of your ASVs have under 50 reads — lots of low-abundance features, again, common in soils.                              |
| Maximum frequenc   | One dominant ASV appears almost 200k times — likely a key core member.                                             |
| Mean frequency      | Average abundance across all ASVs. Skewed by a few highly abundant taxa.                                           |


## Feature Table Visualization (`table.qzv`) — Pages 2 & 3

## Page 2: Interactive Sample Detail

This page provides a searchable, sortable summary of each sample in your ASV table, including:

- **Sample ID**
- **Total frequency (number of reads)**
- **Metadata columns** (from your `metadata.txt` file)

**Use this page to:**
- Identify samples with **very low read counts** (e.g., <10,000), which may be removed before diversity analyses.
- Confirm that your **metadata file matches the sample IDs** from the sequencing run.
- Visually explore **read depth trends across timepoints, treatments, or sites**.

## Page 3: Interactive Feature Detail

This page displays every **ASV (feature)** in the table with:

- **Feature ID**
- **Total frequency (how many reads it has across all samples)**
- **Prevalence** (number of samples it appears in)

**Use this page to:**
- Identify **dominant ASVs** that appear in many samples or with high abundance.
- Flag **rare ASVs** (e.g., features found in only 1 sample or <10 reads) for potential filtering.
- Export ASV tables for downstream filtering, taxonomic assignment, or visualization.

Visualize rep-seqs 
```
qiime feature-table tabulate-seqs \
--i-data rep-seqs.qza \
--o-visualization rep-seqs.qzv
```
This visualization shows all **unique ASV sequences** inferred during denoising (via DADA2), along with:

- **Feature ID** — the hash-like identifier (e.g., `f47c1c7b1f0a2f73...`)
- **Sequence** — the actual 16S rRNA gene fragment (nucleotide sequence), you can use this in BLAST if needed
- **Sequence length** — number of base pairs (should be consistent if region was trimmed uniformly), (scroll all the way to the right)

## 6. Assign Taxonomy to Reads

⚠️ **Why GTDB alone is not enough for amplicon data:**

GTDB is optimized for **whole-genome phylogeny of bacteria and archaea**. It was **not designed to classify 16S reads** that might include **chloroplasts, mitochondria, or eukaryotes**.

When used with QIIME2's `feature-classifier classify-sklearn`, GTDB doesn’t flag these sequences as “unassigned.” Instead, it **forces them into the closest bacterial match**, which can lead to inaccurate classifications:

- **Chloroplasts** (from plant DNA) → misclassified as **Cyanobacteriota**
- **Mitochondria** → misclassified as **Alphaproteobacteria**
- **Eukaryotic reads** → randomly misassigned to bacterial groups like **Planctomycetota** or **Verrucomicrobiota**
- Some appear as **`Unclassified` or `d__Bacteria;__;__;...`**, but not reliably

## **Best Practice Workflow**  
This hybrid approach ensures you're not falsely inflating bacterial diversity by misclassifying organelles or eukaryotic reads, while still aligning with GTDB-based MAG analysis downstream.

If you're working with both **ASVs and MAGs**, and you want to use GTDB for consistency in MAG taxonomy:

A. **First classify ASVs using SILVA**
   - SILVA accurately identifies and labels **chloroplasts**, **mitochondria**, and **eukaryotes**

B. **Filter out unwanted taxa** (chloroplasts, mitochondria, euks) from your SILVA-classified table

C. **Join the cleaned ASV list** with GTDB classifications using **ASV ID** as the key in R
   - This allows you to keep **GTDB taxonomy for downstream analysis** without retaining contamination

D. **Keep a copy of the merged table** (e.g., `taxonomy-merged-silva-gtdb.txt`) in case you need to trace or validate assignments later

## Silva classification
```
#!/bin/bash
#SBATCH --nodes=1
#SBATCH --ntasks=4
#SBATCH --time=4-00:00:00
#SBATCH --mail-type=BEGIN,END,FAIL
#SBATCH --mem=20gb
#SBATCH --mail-user=First.Last@colostate.edu
#SBATCH --mail-type=BEGIN,END,FAIL
#SBATCH --partition=wrighton-hi,wrighton-low
#SBATCH --job-name=Silva138

source /home/opt/Miniconda3/miniconda3/bin/activate qiime2-2023.9

qiime feature-classifier classify-sklearn \
--i-classifier /home/Database/qiime2/classifiers/qiime2-2023.9/silva-138-99-515-806-nb-classifier.qza \
--i-reads rep-seqs.qza \
--o-classification taxonomy_silva138.qza
```
## GTDB classification
```
#!/bin/bash
#SBATCH --nodes=1
#SBATCH --ntasks=4
#SBATCH --time=4-00:00:00
#SBATCH --mail-type=BEGIN,END,FAIL
#SBATCH --mem=20gb
#SBATCH --nodelist=zenith
#SBATCH --mail-user=eryn.grant@colostate.edu
#SBATCH --mail-type=BEGIN,END,FAIL
#SBATCH --partition=wrighton-hi,wrighton-low
#SBATCH --job-name=GTDB220

source /home/opt/Miniconda3/miniconda3/bin/activate qiime2-2023.9

qiime feature-classifier classify-sklearn \
--i-classifier /home/Database/qiime2/classifiers/qiime2-2023.9/GTDBclassifier220_EMP.qza \
--i-reads rep-seqs.qza \
--o-classification taxonomy_gtdb_220.qza
```
Visualize taxonomy table
```
qiime metadata tabulate \
--m-input-file taxonomy_gtdb_220.qza \
--o-visualization taxonomy_gtdb_2204.qzv
```
You can also make stacked taxonomy plots from this step
```
qiime taxa barplot \
  --i-table table.qza \
  --i-taxonomy taxonomy_silva138.qza \
  --m-metadata-file metadata.txt \
  --o-visualization taxa-barplot_silva138.qzv
```
## 7. Filter out Chloroplast and Mitochondria reads
```
qiime taxa filter-table \
--i-table table.qza \
--i-taxonomy taxonomy_silva138.qza \
--p-exclude mitochondria,chloroplast,Unassigned,Eukaryota \
--o-filtered-table taxonomy_silva138_CLEAN.qza
```
Notes:
- --p-exclude matches **case-insensitively** to taxonomy strings.

Visualize clean taxonomy table to double check that your filtering step worked correctly
```
qiime metadata tabulate \
--m-input-file taxonomy_silva138_CLEAN.qza \
--o-visualization taxonomy_silva138_CLEAN.qzv
```

## 8. Convert raw feature table to text file
This will produce a .biom file in a new folder. The new folder will have a name that is a string of letters and numbers.  
Ex: 105a399a-1af3-42df-a83e-e2a23a03a103  
You will need to go into this folder and then into a folder named `data`. Once there, you will find the `feature-table.biom`. depending on your downstream workflow you can use the .biom format or you can convert it to a .txt file. 

|Tool / Library     | Accepts `.biom`? | Preferred Format | Notes | 
| ------------------|------------------|------------------|------------------------------------------------------------- |
| [QIIME2](https://qiime2.org)            | ✅ Yes           |  `.biom`         | Native format                                                |
| [Phyloseq (R)](https://joey711.github.io/phyloseq/)      | ✅ Yes           | `.biom`          | Import .biom files directly `use import_biom())`             |
| [MicrobiomeAnalyst](https://www.microbiomeanalyst.ca/) | ✅ Yes           | `.biom`, `.tsv`  | Web-based — accepts `.biom` or `.tsv`                        |
| [LEfSe/ MaAsLin2 (Galaxy)](https://huttenhower.sph.harvard.edu/galaxy/)    | ❌ No            | `.txt`           | LEfSe-specific format                                        |

Convert from .biom to .txt

```
unzip taxonomy_silva138_CLEAN.qza
```
```
biom convert -i feature-table.biom -o feature-table.txt --to-tsv
```

---

📅 **Note on Multi-Year Data**

This project includes two separate years of amplicon sequencing data.  
For downstream analysis, the datasets are processed separately through denoising (DADA2),  
then **merged prior to taxonomy classification** to ensure consistent ASV tracking.

See the section on **merging multiple datasets** for details on how to combine `table.qza` and `rep-seqs.qza` files across years.





