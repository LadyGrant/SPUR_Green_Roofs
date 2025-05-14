# Rooftop Agrivoltaic Microbiomes Project

This repository contains workflows, metadata, and documentation for a multi-year study investigating microbial communities in urban rooftop agrivoltaic agroecosystems. 

---

## Project Summary

We analyzed soil microbiomes from rooftop garden plots over **two growing seasons** (2023 and 2024), comparing:

- Shaded (agrivoltaic) vs. full sun green roof systems
- Rhizosphere vs. bulk soils
- “Starting” greenhouse microbiomes vs. transplanted systems
- Green roof (rooftop) vs. ground-level beds
- Planting vs. harvest timepoints

Amplicon sequencing of 16S rRNA genes was used alongside soil chemical and enzyme assays to explore patterns in microbial community composition and function.

---

## Data Types

- 16S rRNA amplicon sequencing (2 years)
- Soil enzyme activity (e.g., β-glucosidase)
- Soil chemical properties (OM%, C:N, NO₃⁻, etc.)
- Metadata (site, timepoint, treatment, etc.)

---

## Workflow Documentation

- [QIIME2 Workflow (Amplicon Processing)](qiime2_workflow.md)

This document includes detailed steps for:
- Importing and demultiplexing reads
- Running DADA2 denoising
- Taxonomy classification (SILVA + GTDB)
- Filtering organelle/eukaryotic reads
- Merging multi-year datasets





