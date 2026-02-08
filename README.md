

**ASSIGNMENT 1**
*in progress*

**Introduction**

The objective of this project is to assemble a *Salmonella enterica* genome from Oxford Nanopore Technologies (ONT) long reads (R10 chemistry), then compare the resulting consensus assembly to a public reference genome from NCBI to identify and visualize variants.

The goal of genome assembly is described as producing complete, accurate sequences with minimal fragmentation. This has been described as the “one chromosome, one contig,” ideal (Koren & Phillippy, 2015).  Long reads help span repeats and structural complexity that commonly break short-read assemblies, but ONT reads historically have higher per-base error rates than Illumina, which can leave small errors that matter for downstream interpretation and variant calling (Wick & Holt, 2021). The choice of tools used in downstream steps therefore must be chosen in a way that best mitigates these errors, with quality control being a consistent consideration. 

**In assembling a genome,** there are a number of tools that are used in literature. Benchmarks on prokaryotic long-read data consistently find that there are several top performing assembly tools across different metrics, such as Flye (Kolmogorov et al., 2019). However, no single assembler performs perfectly on every metric (Wick & Holt, 2021). 

Comparing genetic material collected from a sample of an organism to a reference genome of the same organism serves as a way to identify variation. The challenge in this lies in quantifying variants as being due to biological differences versus errant differences. Raw reads are preferred when mapping against the reference genome, as opposed to the assembly, since it offers a more direct comparison when variant calling (?). The variable performance of assemblers also means that mapping assembled genomes to the reference could lead to variants being called could be errant calls due to missassembly errors and is only subverted when the genome assembly is of perfect quality (Rojas-Miranda et al., 2025)

Minimap2 is an alignment tool designed for ONT reads (Li, 2018). Minimap2 is stated to be a generally good option in literature, for long reads.

The variant calling step is an integral part in understanding what differences exist between the reference genome and the freshly sequenced reads from our sample: it helps us understand if discrepancies between our sample and the reference were errors made in the process of sequencing, assembly and processing, or if they are biologically significant variations in genetic sequence. Among the tools that exist for this purpose, variant calling through recent, deep learning driven algorithms like Clair3 seem to outperform traditional tools (Hall et al., 2024)

As a final step, in order to visualize the findings of the project, IGV is a standard tool for interactive genome data exploration (Thorvaldsdottir et al., 2012).

**Methods:**

### **Data and project organization**

Oxford Nanopore Technologies (ONT) **R10** whole-genome sequencing reads (FASTQ) were used to assemble a *Salmonella enterica* genome and compare it to a public NCBI reference via read mapping and variant calling. The overall workflow followed: **QC → assembly → assembly QC → read alignment to reference → variant calling → visualization**.

**Directory structure (key outputs):**

* `01_reads/` raw reads (`*.fastq.gz`)

* `00_refGenome/` NCBI reference FASTA \+ GFF3 annotation

* `02_qc/fastqc/` FastQC HTML reports

* `03_assembly/flye/` Flye assembly (`assembly.fasta`) 
* `04_quast/quast_flye/` QUAST report \+ plots

* `05_align/` sorted/indexed BAM alignments

* `06_variant/clair3/` VCF outputs from Clair3

* `figures/` exported plots/screenshots 

* `logs/` saved command logs (`flye.log`, `quast_flye.log`, `clair3.log`.)

### **Reference genome download (NCBI Datasets CLI)**

A reference assembly for *S. enterica* (accession **GCF\_000006945.2**) was downloaded using NCBI Datasets, including the reference genome (FASTA) and annotations (GFF3).


`datasets download genome accession GCF_000006945.2 --include genome,gff3 --filename ncbi_dataset.zip`  
`unzip -q ncbi_dataset.zip`

### **Read quality control (FastQC)**

Raw reads were assessed with FastQC to summarize per-base quality, GC content, and read length distributions.

`fastqc -t 8 -o 02_qc/fastqc 01_reads/SRR32410565*.fastq`


### **Genome assembly (Flye)**

Reads were assembled using Flye with the ONT high-quality preset (`--nano-hq`). Flye was executed via Docker for reproducibility.

`docker run --rm \`  
  `-u "$(id -u)":"$(id -g)" \`  
  `-v "$PWD":/work -w /work \`  
  `nanozoo/flye:2.9.6--8f3dad7 \`  
  `flye \`  
    `--nano-hq 01_reads/SRR32410565.fastq.gz \`  
    `--read-error 0.03 \`  
    `--threads 10 \`  
    `--out-dir 03_assembly/flye \`  
  `|& tee logs/flye.log`

### **Assembly evaluation (QUAST)**

QUAST was used to evaluate contiguity and agreement with the reference genome and annotation, producing summary metrics and plots (including Circos, when enabled).

`docker run --rm \`  
  `-u "$(id -u)":"$(id -g)" \`  
  `-v "$PWD":/work -w /work \`  
  `staphb/quast:5.2.0-slim \`  
  `quast.py 03_assembly/flye/assembly.fasta \`  
    `-r 00_refGenome/GCF_000006945.2_ASM694v2_genomic.fna \`  
    `-g 00_refGenome/genomic.gff \`  
    `--circos \`  
    `-o 04_quast/quast_flye \`  
    `-t 10 \`  
  `|& tee logs/quast_flye.log`

### **Read alignment to the reference (minimap2 \+ samtools)**

Raw reads were aligned to the reference genome using minimap2 (long-read mapping preset for HQ reads) and processed into a coordinate-sorted, indexed BAM for downstream variant calling and visualization. Sorted/indexed BAM is required for IGV.

`THREADS=10`  
`REF="00_refGenome/GCF_000006945.2_ASM694v2_genomic.fna"`  
`READS="01_reads/SRR32410565.fastq.gz"`

`minimap2 -ax lr:hq -t "$THREADS" "$REF" "$READS" \`  
  `| samtools view -b -o 05_align/reads_to_ref.bam -`

`samtools sort -@ "$THREADS" -o 05_align/reads_to_ref.sorted.bam 05_align/reads_to_ref.bam`  
`samtools index 05_align/reads_to_ref.sorted.bam`  
`samtools faidx "$REF"`

### **Variant calling (Clair3)**

Variants (SNPs/indels) were called from the **raw-read alignment to the reference** using Clair3 (ONT model `r1041_e82_400bps_sup_v500`).


`THREADS=10`  
`REF="00_refGenome/GCF_000006945.2_ASM694v2_genomic.fna"`  
`MODEL="/opt/models/r1041_e82_400bps_sup_v500"`

`docker run --rm \`  
  `-u "$(id -u)":"$(id -g)" \`  
  `-v "${PWD}":/work -w /work \`  
  `hkubal/clair3:latest \`  
  `/opt/bin/run_clair3.sh \`  
    `--bam_fn="05_align/reads_to_ref.sorted.bam" \`  
    `--ref_fn="${REF}" \`  
    `--threads="${THREADS}" \`  
    `--platform="ont" \`  
    `--model_path="${MODEL}" \`  
    `--include_all_ctgs \`  
    `--output="06_variant/clair3" \`  
    `--no_phasing_for_fa`

### **Visualization (IGV)**

The sorted BAM (`reads_to_ref.sorted.bam`) and Clair3 VCF (`*.vcf.gz` \+ index) were loaded into IGV for interactive inspection of variant evidence (read pileups, allele ratios) and gene context using the reference annotation track.

---


---

## **Discussion**


### **Limitations and future directions**

A key limitations of this analysis would be the dependence on a single chosen reference assembly. A practical next step for improved consensus accuracy would be to apply an ONT-focused polishing stage (for eg. Medaka or Racon) prior to downstream analyses, then re-check whether candidate variants are still present.

# **References**

Hall, M. B., Wick, R. R., Judd, L. M., Nguyen, A. N., Steinig, E. J., Xie, O., Davies, M., Seemann, T., Stinear, T. P., & Coin, L. (2024). Benchmarking reveals superiority of deep learning variant callers on bacterial nanopore sequence data. *ELife*, *13*. https://doi.org/10.7554/elife.98300

Kolmogorov, M., Yuan, J., Lin, Y., & Pevzner, P. A. (2019). Assembly of long, error-prone reads using repeat graphs. *Nature Biotechnology*, *37*(5), 540–546. https://doi.org/10.1038/s41587-019-0072-8  
Koren, S., & Phillippy, A. M. (2015). One chromosome, one contig: complete microbial genomes from long-read sequencing and assembly. *Current Opinion in Microbiology*, *23*, 110–120. https://doi.org/10.1016/j.mib.2014.11.014  
Lee, J. Y., Kong, M., Oh, J., Lim, J., Chung, S. H., Kim, J.-M., Kim, J.-S., Kim, K.-H., Yoo, J.-C., & Kwak, W. (2021). Comparative evaluation of Nanopore polishing tools for microbial genome assembly and polishing strategies for downstream analysis. *Scientific Reports*, *11*(1). https://doi.org/10.1038/s41598-021-00178-w  
Li, H. (2018). Minimap2: pairwise alignment for nucleotide sequences. *Bioinformatics*, *34*(18), 3094–3100. https://doi.org/10.1093/bioinformatics/bty191  
*nanoporetech/medaka*. (2021, April 15). GitHub. https://github.com/nanoporetech/medaka  
Thorvaldsdottir, H., Robinson, J. T., & Mesirov, J. P. (2012). Integrative Genomics Viewer (IGV): high-performance genomics data visualization and exploration. *Briefings in Bioinformatics*, *14*(2), 178–192. https://doi.org/10.1093/bib/bbs017  
Wick, R. R., & Holt, K. E. (2021). Benchmarking of long-read assemblers for prokaryote whole genome sequencing. *F1000Research*, *8*, 2138\. https://doi.org/10.12688/f1000research.21782.4  
Wick, R. R., Howden, B. P., & Stinear, T. P. (2025). Autocycler: long-read consensus assembly for bacterial genomes. *Bioinformatics*, *41*(9). https://doi.org/10.1101/2025.05.12.653612  
Wick, R. R., Judd, L. M., & Holt, K. E. (2023). Assembling the perfect bacterial genome using Oxford Nanopore and Illumina sequencing. *PLOS Computational Biology*, *19*(3), e1010905. https://doi.org/10.1371/journal.pcbi.1010905

Hall, M. B., Wick, R. R., Judd, L. M., Nguyen, A. N., Steinig, E. J., Xie, O., Davies, M., Seemann, T., Stinear, T. P., & Coin, L. (2024). Benchmarking reveals superiority of deep learning variant callers on bacterial nanopore sequence data. *ELife*, *13*. https://doi.org/10.7554/elife.98300  
Rojas-Miranda, H., Madrigal-Ly, V., & Molina-Mora, J. A. (2025). Benchmarking genome assemblers for four bacterial models based on contiguity, correctness, and completeness. *Scientific Reports*, *15*(1). https://doi.org/10.1038/s41598-025-26847-8

