# Synthetic haematology panel dataset and pipeline validation
 
<!-- Replace <user>/<repo> in the badge link once the repo exists. -->
[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/leongriner-portfolio/synth-haem/blob/main/synth_haem.ipynb)
 
<!-- 2-3 sentences, your own words: what this is, what it measures, the headline result.
     Example shape: "A synthetic test dataset for a 150-gene haematology capture panel,
     with 21 known variants spiked in at 2-50% VAF, used to measure a tumour-only
     variant-calling pipeline's sensitivity by VAF tier and its false-positive rate."
     Add the headline number here once you have it. -->
 
**Status:** <!-- e.g. "In progress: dataset built, pipeline scored" / "Complete" -->
 
## The question
 
<!-- One or two sentences. Suggested: Does a tumour-only pipeline find clinically relevant
     haematology variants at low allele fractions on this panel, without making false calls? -->
 
## Results
 
<!-- Fill from step 9's output (output/results/eval/). Put this section near the top:
     reviewers look here first. -->
 
| Tier (VAF) | SNVs found | Indels found | False positives |
|---|---|---|---|
| 40% | _ of 3 | _ of 1 | — |
| 10% | _ of 3 | _ of 1 | — |
| 5% | _ of 3 | _ of 1 | — |
| 2% | _ of 4 | _ of 1 | — |
| 50% (germline-like) | _ of 3 | _ of 1 | — |
| Sample A (negative) | — | — | _ |
| Sample B, outside the truth set | — | — | _ |
 
<!-- One short paragraph: what was missed and why (e.g. a 2% variant with few supporting
     reads), and what the false positives were. Link the per-variant table:
     output/results/eval/B_results_by_variant.tsv -->
 
## Design
 
### Panel
 
- **Gene list:** the 150 genes publicly listed for Peter MacCallum Cancer Centre's
  [AllHaem-CGP DNA panel](https://www.petermac.org/health-professionals/services-for-health-professionals/pathology-health-professionals/molecular-pathology/molecular-haematology).
  This is a proxy built from that public list; the project has no affiliation with Peter Mac.
- **Capture design:** Agilent SureSelect Cancer CGP Design Wizard, hg38.
- **Reference:** GIAB's masked GRCh38 (no-alt analysis set, GRC false duplications
  replaced by N). This fixes the false U2AF1 duplication on chr21 described by
  [Miller et al. 2022](https://pmc.ncbi.nlm.nih.gov/articles/PMC8950341/); the nine design
  rows on the false copy were removed.
| File (after cleaning) | Regions | Bases | Used for |
|---|---|---|---|
| `Regions_SNV.clean.bed` | 2,170 | 380,700 | Calling and scoring |
| `Covered_SNV.clean.bed` | 2,068 | 485,357 | Baits; read source after 200 bp padding |
| `panel_src.bed` (padded, merged) | 1,600 | 1,206,514 | Where reads are simulated from |
 
### Dataset
 
- **Sample A (negative):** 2×150 reads simulated with ART (HiSeq X profile, seed 42)
  from the padded panel regions, about 1000x. No variants.
- **Sample B (positive):** Sample A's reads with 21 variants spiked in by
  BAMSurgeon (commit d9448a5), each at a set allele fraction.
- **Truth VCF:** what BAMSurgeon reports actually landed.
| Tier (VAF) | Variants | What it tests |
|---|---|---|
| 40% | JAK2 V617F, SF3B1 K700E, TP53 R273H, ASXL1 G646Wfs\*12 | SNVs; homopolymer indel |
| 10% | IDH1 R132H, SRSF2 P95H, MYD88 L265P, CALR type 1 (52 bp deletion) | GC-rich exon; large deletion |
| 5% | IDH2 R140Q, BRAF V600E, U2AF1 S34F, FLT3-ITD (30 bp) | The masked reference; tandem duplication |
| 2% | NRAS G12D, KRAS G12D, DNMT3A R882H, RHOA G17V, NPM1 type A | Low-VAF sensitivity |
| 50% | DDX41 D140Gfs\*2, TP53 P72R, TET2 I1762V, FLT3 T227M | Germline-like variants in tumour-only calling |
 
<!-- Optional: one line on why TET2 I1762V and FLT3 T227M (common benign SNPs,
     gnomAD frequency 0.30 and 0.61) and why DDX41 (germline predisposition). -->
 
### Test pipeline
 
BWA-MEM → GATK MarkDuplicates → GATK Mutect2 (tumour-only, targets +50 bp) →
FilterMutectCalls → PASS calls, normalised with bcftools → scored against the
truth VCF with RTG vcfeval (`--squash-ploidy`, inside the targets). Results table in base R.
 
<!-- One sentence on why tumour-only: B's background reads are A's, so A as a matched
     normal would cancel every artefact perfectly. -->
 
## Limitations
 
<!-- Keep these: stating them is part of a credible validation. -->
 
- **Even coverage:** ART samples uniformly, so there is no GC or probe bias.
- **No off-target reads:** on-target rate is unrealistically high.
- **No PCR duplicates or UMIs.**
- **Almost no germline background:** only the four variants at 50%.
- **No copy number, structural variants or IG/TCR rearrangements.**
<!-- Planned next steps, if any: GIAB HG002 check, second seed, CNV sample. -->
 
## How to reproduce
 
Runs on free Google Colab (2 CPUs, ~12.7 GB RAM) with Google Drive for storage.
<!-- Add measured times once you have them. -->
 
1. Make a project folder in your Google Drive and set `PROJECT` in notebook 01's
   first cell to its path.
2. Put the design files in `reference/sureselect/Haem_1/` in that folder.
   <!-- Or explain how to obtain them, if you don't publish them. -->
3. Open [notebook 01](notebooks/01_environment_setup.ipynb) in Colab and run
   sections 1–9: this builds the conda environment from `env/bioinf_portfolio.lock.yml`.
4. Open [notebook 02](notebooks/02_simulated_dataset.ipynb) and run its setup
   cell and steps 0–9. Save to Drive (notebook 01, section 11) after the long steps.
Drive needs about 13 GB free: the reference and its indexes take about 10 GB.
The full run is several hours of compute, best split over three sessions
(see notebook 02's session plan).
 
## Repository layout
 
```
├── notebooks/             Colab notebooks (01 setup, 02 dataset and scoring)
├── env/                   environment.yml and the exact lock file
├── reference/sureselect/  cleaned BEDs and interval lists
├── data/raw/truth/        spike-in list, spike.vcf, truth VCF
├── data/raw/sim/          read-source regions (FASTQs are not in git; see below)
├── data/processed/calls/  pipeline VCFs
└── output/                QC, scoring, results table, tool versions
```
 
## Data availability
 
The FASTQs (~2 GB) are not in this repository.
<!-- Pick one:
     - "They are on Zenodo: <DOI>."
     - "They can be regenerated exactly: every step uses fixed seeds (ART -rs 42,
        BAMSurgeon --seed 42)." -->
 
## Software
 
Recorded by notebook 01 (`output/logs/tool_versions.txt`); the exact environment is
in `env/bioinf_portfolio.lock.yml`.
 
| Tool | Version |
|---|---|
| Python | 3.11.18 |
| micromamba | 2.10.0 |
| samtools, bcftools, htslib | 1.24 |
| bedtools | 2.31.1 |
| BWA | 0.7.19-r1273 |
| ART | 2016.06.05 (Q Version 2.5.8) |
| mosdepth | 0.3.14 |
| Picard | 3.5.0 |
| GATK | 4.6.2.0 |
| RTG Tools | 3.13 |
| OpenJDK | 17.0.18 |
| R | 4.6.2 |
| BAMSurgeon | commit d9448a5 (GitHub, installed with pip) |
 
## References
 
<!-- Add or trim as your write-up needs. -->
 
- Miller CA et al. (2022). The false U2AF1 duplication in GRCh38. <https://pmc.ncbi.nlm.nih.gov/articles/PMC8950341/>
- Genome in a Bottle mapping references. <https://ftp-trace.ncbi.nlm.nih.gov/ReferenceSamples/giab/release/references/README_GIAB_Mapping_References.md>
- BAMSurgeon. <https://github.com/adamewing/bamsurgeon>
- ART read simulator. <https://www.niehs.nih.gov/research/resources/software/biostatistics/art>
- GATK. <https://gatk.broadinstitute.org/>
- RTG Tools. <https://github.com/RealTimeGenomics/rtg-tools>
- Roy S et al. (2018). Standards and guidelines for validating next-generation sequencing
  bioinformatics pipelines (AMP/CAP). *J Mol Diagn* 20(1):4–27.
## Licence
 
<!-- e.g. MIT for code (add a LICENSE file); CC BY 4.0 for data on Zenodo. -->
 
## Author
 
<!-- Your name, and a LinkedIn or contact link if you want one. -->
