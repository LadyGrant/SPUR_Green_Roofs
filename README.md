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
#SBATCH --mail-user=eryn.grant@colostate.edu
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

