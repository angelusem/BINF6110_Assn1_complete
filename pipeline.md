**The pipeline followed for the analysis is as follows:**

**Directory structure:**

* 01\_reads/ raw reads (\*.fastq.gz)

* 00\_refGenome/ NCBI reference FASTA \+ GFF3 annotation

* 02\_qc/fastqc/ FastQC HTML reports

* 03\_assembly/flye/ Flye assembly (assembly.fasta) \+ logs

* 04\_quast/quast\_flye/ QUAST report \+ plots

* 05\_align/ sorted/indexed BAM alignments

* 06\_variant/clair3/ VCF outputs from Clair3

* figures/ exported plots/screenshots used in this README (with captions)

* logs/ saved command logs (flye.log, quast\_flye.log, etc.)

### **Reference genome download (NCBI Datasets CLI)**

A curated RefSeq assembly for *S. enterica* (accession **GCF\_000006945.2**) was downloaded using NCBI Datasets, including the reference genome (FASTA) and annotations (GFF3).

`datasets download genome accession GCF_000006945.2 --include genome,gff3 --filename ncbi_dataset.zip`  
`unzip -q ncbi_dataset.zip`

### **Raw reads download (NCBI direct download)**

Found under WGS of Salmonella enterica isolate (SRR32410565) entry in Sequence Read Archive: [https://trace.ncbi.nlm.nih.gov/Traces/?run=SRR3241056](https://trace.ncbi.nlm.nih.gov/Traces/?run=SRR3241056)

### **Read quality control (FastQC**:version 0.12.1**)** 

Raw reads were assessed with FastQC (Andrews, 2010\) to summarize per-base quality, GC content, and read length distributions.

`fastqc -t 8 -o 02_qc/fastqc 01_reads/SRR32410565*.fastq`

### **Genome assembly (Flye:**version 2.9.6**)**

Reads were assembled using Flye with the ONT high-quality preset (--nano-hq). Flye was executed via Docker for reproducibility.

`docker run --rm \`  
  `-u "$(id -u)":"$(id -g)" \`  
  `-v "$PWD":/work -w /work \`  
  `nanozoo/flye:2.9.6--8f3dad7 \`  
  `flye \`  
    `--nano-hq 01_reads/SRR32410565.fastq.gz \`  
    `--read-error 0.03 \`  
    `--threads 8 \`  
    `--out-dir 03_assembly/flye \`  
  `|& tee logs/flye.log`

### **Assembly evaluation (QUAST:**version 5.2.0**)**

QUAST was used to evaluate contiguity and agreement with the reference genome and annotation, producing summary metrics and plots. 

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
`#for the circos plot:`  
`docker run --rm \`  
  `-u "$(id -u)":"$(id -g)" \`  
  `-v "$PWD":/input \`  
  `-v "$PWD":/work -w /work \`  
  `erasche/circos \`  
  `-conf 04_quast/quast_flye/circos/circos.conf`

### **Read alignment to the reference and pre-processing for visualization (minimap2:**version 2.30 and **samtools:**version 1.23**)**

Raw reads were aligned to the reference genome using minimap2 and processed into a coordinate-sorted, indexed BAM by samtools for downstream variant calling and visualization. 

`THREADS=10`

`REF="00_refGenome/GCF_000006945.2_ASM694v2_genomic.fna"`  
`READS="01_reads/SRR32410565.fastq.gz"`

`minimap2 -ax lr:hq -t "$THREADS" "$REF" "$READS" \`  
  `| samtools view -b -o 05_align/reads_to_ref.bam -`

`samtools sort -@ "$THREADS" -o 05_align/reads_to_ref.sorted.bam 05_align/reads_to_ref.bam`  
`samtools index 05_align/reads_to_ref.sorted.bam`  
`samtools faidx "$REF"`

### **Variant calling (Clair3)**

Variants (SNPs/indels) were called from the **raw-read alignment to the reference** using Clair3 (ONT model r1041\_e82\_400bps\_sup\_v500). 

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

### **Visualization (IGV: version 2.19.7)**

The sorted BAM and Clair3 VCF were loaded into IGV for interactive inspection of variant evidence and gene context using the reference annotation track.
