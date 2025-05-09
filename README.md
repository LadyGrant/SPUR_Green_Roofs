# SPUR rooftop agrivoltaic 16S QIIME2 pipeline
## 1. Import Sequence Files into QIIME2

We used the **515F/806R V4 primers** from the Earth Microbiome Project (EMP): 
 [Earth Microbiome Project protocol](https://earthmicrobiome.org/protocols-and-standards/)

```
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

## 3. Visualize sequence reads and sequence quality
After demultiplexing, it's important to assess the number of reads per sample and the quality of those reads before proceeding with denoising.
```
qiime demux summarize --i-data demux.qza --o-visualization demux.qzv
```
Open the resulting .qzv file in [QIIME2 View](https://view.qiime2.org/) to explore the interactive summary.

# Overview of demux.qzv Summary Tabs

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
<img width="1425" alt="Screenshot 2025-05-08 at 7 57 44 PM" src="https://github.com/user-attachments/assets/3544c0d1-c4be-4ab9-9299-ad871019eb39" />












