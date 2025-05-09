# SPUR_Green_Roofs
1. Import Sequence files into your working directory
we uses the 515/806 V4 primers from the Earth Microbiome Project
Earth Microbiome Project protocol: https://earthmicrobiome.org/protocols-and-standards/
```
qiime tools import --type EMPPairedEndSequences --input-path reads/ --output-path emp-paired-end-sequences.qza 
```

Explanation of Flags:
Flag		   |	Description
__________________________________________________________________________________________
			   |
--type 		   |	Specifies that the input is in Earth Microbiome Project format 
			   |	for paired-end reads (i.e., contains forward.fastq.gz, 
			   |	reverse.fastq.gz, and barcodes.fastq.gz)
_______________|__________________________________________________________________________							  
--input-path   | Path to your raw sequencing directory containing the three FASTQ files
_______________|__________________________________________________________________________
--output-path  | Name of the output artifact that wraps your imported reads in 
			   | QIIME2’s .qza format for further processing
_______________|__________________________________________________________________________ 

2. Demultiplex the sequences
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

Inputs: metadata.txt
outputs: demux.qza, demux-details.qza
