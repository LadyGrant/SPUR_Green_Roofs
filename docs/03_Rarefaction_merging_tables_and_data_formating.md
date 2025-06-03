# Rarefaction, Table Merging, and Data Formatting for 16S Amplicon Analysis in R

This document walks through how to:
- Choose a rarefaction depth (if you are rarefying)
- Merge taxonomy with your ASV count table
- Format and clean the data for downstream analysis

Designed for researchers working with QIIME2 outputs in R with a focus on clarity and reproducibility.

# Note on rarefaction




# Prepping the ASV Table Before Importing into R
Before uploading the .txt file (exported from QIIME2) into R:

Delete the first line that reads:
`# Constructed from biom file`
This is a comment line added by QIIME and will interfere with column alignment in R.

Change the first column header from #OTU ID to ASV_ID
This avoids issues with spaces in column names when using R or tidyverse functions.

These edits ensure smooth import using `read_table()` or `read.delim()` and make downstream data manipulation easier.

First we need to load the needed libraries, set your working directory, set a seed and read in your files.
```
# Load libraries
library(vegan)
library(tidyverse)
library(ggpubr)
set.seed(12345)

# Set working directory
setwd("~/OneDrive - Colostate/01_Agbiome/SPUR_GreenRoofs/16S_R/")

# Read input files
raw <- read_table("Silva138_16S_Merged_23_24_RAW.txt")
metadata <- read_table("metadata2324.txt")
taxonomy <- read.delim("taxonomy.tsv")
```

# Part 1: Identify contaminant ASVs from NTC samples and Buchnera (endosymbiont of aphids).
Buchnera is typically at lower concentations and will not cause any issues but this system had one year had an aphid infestation while the other did not. Since we are already dealing with two sequencing runs and high differences in sequencing depths, we need to drop as much decrepensies etween the years as possilbe while preserving the biological signature. 

Process taxonomy
```
taxonomy_split <- taxonomy %>%
  separate(Taxon, into = c("Domain", "Phylum", "Class", "Order", "Family", "Genus", "Species"), sep = ";", fill = "right") %>%
  rename(ASV_ID = Feature.ID)
```

Merge taxonomy with counts
```
raw_table <- raw %>%
  pivot_longer(2:177, names_to = "SampleID", values_to = "Count")
taxa_long <- left_join(raw_table, taxonomy_split, by = "ASV_ID")
```

Identify NTCs
```
ntc_metadata <- metadata %>% filter(Sample_Type %in% c("NTC_23", "NTC_24"))
ntc_ids <- ntc_metadata$SampleID
real_ids <- metadata %>% filter(!Sample_Type %in% c("NTC_23", "NTC_24")) %>% pull(SampleID)
```

Long format table for filtering
```
asv_long <- raw %>% pivot_longer(-`ASV_ID`, names_to = "SampleID", values_to = "Count") %>% 
  rename(ASV_ID = `ASV_ID`)
```






