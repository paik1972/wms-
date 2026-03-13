# 🦠 Gut Microbiome Analysis Pipeline

> Metagenomic profiling and statistical analysis of gut microbiome using MetaPhlAn4

---

## 📋 Overview

This pipeline performs **shotgun metagenomics** analysis on paired-end FASTQ data to characterize gut microbial communities and identify differentially abundant taxa across sample groups.

```
Raw FASTQ (paired-end)
        │
        ▼
┌───────────────┐
│  MetaPhlAn4   │  ──→  Relative Abundance Table  ──→  PCoA (vegan)  ──→  MaAsLin2
│  (Profiling)  │
└───────────────┘
        │
        ▼
┌───────────────┐
│   StrainPhlAn │  ──→  Read Count Table  ──→  PCoA (vegan)  ──→  ANCOM-BC
│  (Counts)     │                                                    DESeq2
└───────────────┘
```

---

## 📁 Project Structure

```
2.ASM/
├── rawdata/                    # Raw paired-end FASTQ files (65 samples)
│   ├── ASM_01_001_1.fastq.gz
│   ├── ASM_01_001_2.fastq.gz
│   └── ...
│
└── metaphlan4_out/
    ├── bowtie2/                # Bowtie2 alignment files (.bowtie2.bz2)
    ├── profiles/               # Per-sample relative abundance profiles
    ├── profiles_counts/        # Per-sample estimated read counts
    └── merged/
        ├── merged_abundance_table.txt   # Merged relative abundance table
        └── merged_counts_table.txt      # Merged read count table
```

---

## 🧬 Samples

| Group | Sample Prefix | N |
|-------|--------------|---|
| Group 1 | ASM_01 | 15 |
| Group 2 | ASM_02 | 16 |
| Group 3 | ASM_03 | 24 |
| Group 4 | KNU_03 | 10 |
| **Total** | | **65** |

---

## ⚙️ Environment

```bash
# Conda environment
conda activate metaphlan_env

# Tool versions
MetaPhlAn 4.x
Bowtie2 2.5.4
Database: mpa_vJan25_CHOCOPhlAnSGB_202503
```

---

## 🚀 Usage

### STEP 1: MetaPhlAn4 Profiling (Relative Abundance)

```bash
metaphlan \
    $HOME/2.ASM/rawdata/${SAMPLE}_1.fastq.gz,$HOME/2.ASM/rawdata/${SAMPLE}_2.fastq.gz \
    --mapout $HOME/2.ASM/metaphlan4_out/bowtie2/${SAMPLE}.bowtie2.bz2 \
    --db_dir /opt/refs/bacteria/metaphlan_db \
    --nproc 4 \
    --input_type fastq \
    --tax_lev a \
    -o $HOME/2.ASM/metaphlan4_out/profiles/${SAMPLE}_profile.txt
```

### STEP 2: Generate Read Count Table (for ANCOM-BC)

```bash
metaphlan \
    $HOME/2.ASM/metaphlan4_out/bowtie2/${SAMPLE}.bowtie2.bz2 \
    --input_type mapout \
    -t rel_ab_w_read_stats \
    --db_dir /opt/refs/bacteria/metaphlan_db \
    -o $HOME/2.ASM/metaphlan4_out/profiles_counts/${SAMPLE}_counts.txt
```

### STEP 3: Merge Tables

```bash
# Relative abundance table
merge_metaphlan_tables.py \
    ~/2.ASM/metaphlan4_out/profiles/*_profile.txt \
    -o ~/2.ASM/metaphlan4_out/merged/merged_abundance_table.txt

# Read count table
merge_metaphlan_tables.py \
    ~/2.ASM/metaphlan4_out/profiles_counts/*_counts.txt \
    -o ~/2.ASM/metaphlan4_out/merged/merged_counts_table.txt
```

---

## 📊 Downstream Analysis (R)

| Analysis | Input | Purpose |
|----------|-------|---------|
| **PCoA** (vegan) | Relative abundance | Beta diversity visualization |
| **MaAsLin2** | Relative abundance | Multivariable association analysis |
| **ANCOM-BC** | Read counts (integer) | Differential abundance (bias-corrected) |
| **DESeq2** | Read counts (integer) | Differential abundance |

---

## 🔧 Troubleshooting

| Error | Cause | Solution |
|-------|-------|----------|
| `unrecognized arguments: --bowtie2out` | Outdated flag | Use `--mapout` instead |
| `unrecognized arguments: --bowtie2db` | Outdated flag | Use `--db_dir` instead |
| `invalid choice: bowtie2out` | Outdated flag | Use `--input_type mapout` |
| `Input file not found: ~/path` | `~` not expanded in comma-joined paths | Use `$HOME` instead of `~` |

---

## 📚 References

- [MetaPhlAn4 Documentation](https://github.com/biobakery/MetaPhlAn)
- [MaAsLin2](https://github.com/biobakery/Maaslin2)
- [ANCOM-BC](https://github.com/FrederickHuangLin/ANCOMBC)
- [DESeq2](https://bioconductor.org/packages/release/bioc/html/DESeq2.html)

---

## 👤 Author

**Kyujin Paik**  
Microbiome Bioinformatics Pipeline  
2026
