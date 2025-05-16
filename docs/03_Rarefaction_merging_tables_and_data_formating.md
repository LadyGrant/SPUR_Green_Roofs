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

Change the first column header from #OTU ID to #ASV_ID (or just ASV_ID)
This avoids issues with spaces in column names when using R or tidyverse functions.

These edits ensure smooth import using `read_table()` or `read.delim()` and make downstream data manipulation easier.

Read in your .txt raw counts file
```
raw <- read_table("Silva138_16S_Merged_23_24.txt")
```

This makes data sets longer by increasing the number of rows and decreasing the number of columns. You should have 3 columns when done: #ASV_ID, Sample, counts.
```
raw_table <- raw %>% 
  pivot_longer(2:268, names_to = "Sample", values_to = "counts")
```


