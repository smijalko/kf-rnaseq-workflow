# Kids First RNA-Seq Workflow

This is the Kids First RNA-Seq pipeline, which includes fusion and expression detection.

![data service logo](https://github.com/d3b-center/d3b-research-workflows/raw/master/doc/kfdrc-logo-sm.png)





## Introduction
This pipeline utilizes cutadapt to trim adapters from the raw reads, if necessary, and passes the reads to STAR for alignment.
The alignment output is used by RSEM for gene expression abundance estimation.
Additionally, Kallisto is used for quantification, but uses pseudoalignments to estimate the gene abundance from the raw data.
Fusion calling is performed using Arriba and STAR-Fusion detection tools on the STAR alignment outputs.
Metrics for the workflow are generated by RNA-SeQC.
A new feature added as of February 27, 2020, is an added workflow to run [annoFuse](https://www.biorxiv.org/content/10.1101/839738v1).
Developed at the Center for Data Driven Discovery in Biomedicine (D3b) by Dr. Jo Lynne Rokita and Krutika Gaonkar, this package adds annotation to arriba results, and some basic filtering, described in the paper.
There is additional filtering capabilities provided beyond the lightweight workflow described in the [D3b annoFuse Workflow](#d3b-annofuse-workflow) section

### Cutadapt
[Cutadapt v2.5](https://github.com/marcelm/cutadapt) Cut adapter sequences from raw reads if needed.
### STAR
[STAR v2.6.1d](https://doi.org/f4h523) RNA-Seq raw data alignment.
### RSEM
[RSEM v1.3.1](https://doi:10/cwg8n5) Calculation of gene expression.
### Kallisto
[Kallisto v0.43.1](https://doi:10.1038/nbt.3519) Raw data pseudoalignment to estimate gene abundance.
### STAR-Fusion
[STAR-Fusion v1.5.0](https://doi:10.1101/120295) Fusion detection for `STAR` chimeric reads.
### Arriba
[Arriba v1.1.0](https://github.com/suhrig/arriba/) Fusion caller that uses `STAR` aligned reads and chimeric reads output.
### RNA-SeQC
[RNA-SeQC v2.3.4](https://github.com/broadinstitute/rnaseqc) Generate metrics such as gene and transcript counts, sense/antisene mapping, mapping rates, etc


## Usage

### Inputs common:
```yaml
inputs:
  sample_name: string
  r1_adapter: {type: ['null', string]}
  r2_adapter: {type: ['null', string]}
  STAR_outSAMattrRGline: string
  STARgenome: File
  RSEMgenome: File
  reference_fasta: File
  gtf_anno: File
  FusionGenome: File
  runThread: int
  RNAseQC_GTF: File
  kallisto_idx: File
  wf_strand_param: {type: [{type: enum, name: wf_strand_param, symbols: ["default", "rf-stranded", "fr-stranded"]}], doc: "use 'default' for unstranded/auto, 'rf-stranded' if read1 in the fastq read pairs is reverse complement to the transcript, 'fr-stranded' if read1 same sense as transcript"}
  input_type: {type: [{type: enum, name: input_type, symbols: ["BAM", "FASTQ"]}], doc: "Please select one option for input file type, BAM or FASTQ."}

```

### Bam input-specific:
```yaml
inputs:
    reads1: File

```

### Fastq input-specific:
```yaml
inputs:
  reads1: File
  reads2: File

```

### Run:

1) For fastq or bam input, run `kfdrc-rnaseq-wf` as this can accept both file types.
For fastq input, please enter the reads 1 file in `reads1` and the reads 2 file in `reads2`.
For bam input, please enter the reads file in `reads1` and leave `reads2` empty as it is optional. 

2) `r1_adapter` and `r2_adapter` are OPTIONAL. 
If the input reads have already been trimmed, leave these as null and cutadapt step will simple pass on the fastq files to STAR. 
If they do need trimming, supply the adapters and the cutadapt step will trim, and pass trimmed fastqs along.

3) `wf_strand_param` is a workflow convenience param so that, if you input the following, the equivalent will propagate to the four tools that use that parameter:
    - `default`: 'rsem_std': null, 'kallisto_std': null, 'rnaseqc_std': null, 'arriba_std': null. This means unstranded or auto in the case of arriba.
    - `rf-stranded`: 'rsem_std': 0, 'kallisto_std': 'rf-stranded', 'rnaseqc_std': 'rf', 'arriba_std': 'reverse'.  This means if read1 in the input fastq/bam is reverse complement to the transcript that it maps to.
    - `fr-stranded`: 'rsem_std': 1, 'kallisto_std': 'fr-stranded', 'rnaseqc_std': 'fr', 'arriba_std': 'yes'. This means if read1 in the input fastq/bam is the same sense (maps 5' to 3') to the transcript that it maps to.

4) Suggested `STAR_outSAMattrRGline`, with **TABS SEPARATING THE TAGS**,  format is:
    
    `ID:sample_name LB:aliquot_id   PL:platform SM:BSID` for example `ID:7316-242   LB:750189 PL:ILLUMINA SM:BS_W72364MN`
5) Suggested inputs are:

    - `FusionGenome`: [GRCh38_v27_CTAT_lib_Feb092018.plug-n-play.tar.gz](https://data.broadinstitute.org/Trinity/CTAT_RESOURCE_LIB/__genome_libs_StarFv1.3/GRCh38_v27_CTAT_lib_Feb092018.plug-n-play.tar.gz)
    - `gtf_anno`: gencode.v27.primary_assembly.annotation.gtf, location: ftp://ftp.ebi.ac.uk/pub/databases/gencode/Gencode_human/release_27/gencode.v27.primary_assembly.annotation.gtf.gz, will need to unzip
    - `RNAseQC_GTF`: gencode.v27.primary_assembly.RNAseQC.gtf, built using `gtf_anno` and following build instructions [here](https://github.com/broadinstitute/rnaseqc#usage)
    - `RSEMgenome`: RSEM_GENCODE27.tar.gz, built using the `reference_fasta` and `gtf_anno`, following `GENCODE` instructions from [here](https://deweylab.github.io/RSEM/README.html), then creating a tar ball of the results.
    - `STARgenome`: STAR_GENCODE27.tar.gz, created using the star_genomegenerate.cwl tool, using the `reference_fasta`, `gtf_anno`, and setting `sjdbOverhang` to 100
    - `reference_fasta`: [GRCh38.primary_assembly.genome.fa](ftp://ftp.ebi.ac.uk/pub/databases/gencode/Gencode_human/release_27/GRCh38.primary_assembly.genome.fa.gz), will need to unzip 
    - `kallisto_idx`: gencode.v27.kallisto.index, built from gencode 27 trascript fasta: ftp://ftp.ebi.ac.uk/pub/databases/gencode/Gencode_human/release_27/gencode.v27.transcripts.fa.gz, following instructions from [here](https://pachterlab.github.io/kallisto/manual)

### Outputs:
```yaml
outputs:
  cutadapt_stats: {type: File, outputSource: cutadapt/cutadapt_stats} # only if adapter supplied
  STAR_transcriptome_bam: {type: File, outputSource: star/transcriptome_bam_out}
  STAR_sorted_genomic_bam: {type: File, outputSource: samtools_sort/sorted_bam}
  STAR_sorted_genomic_bai: {type: File, outputSource: samtools_sort/sorted_bai}
  STAR_chimeric_bam_out: {type: File, outputSource: samtools_sort/chimeric_bam_out}
  STAR_chimeric_junctions: {type: File, outputSource: star_fusion/chimeric_junction_compressed}
  STAR_gene_count: {type: File, outputSource: star/gene_counts}
  STAR_junctions_out: {type: File, outputSource: star/junctions_out}
  STAR_final_log: {type: File, outputSource: star/log_final_out}
  STAR-Fusion_results: {type: File, outputSource: star_fusion/abridged_coding}
  arriba_fusion_results: {type: File, outputSource: arriba_fusion/arriba_fusions}
  arriba_fusion_viz: {type: File, outputSource: arriba_fusion/arriba_pdf}
  RSEM_isoform: {type: File, outputSource: rsem/isoform_out}
  RSEM_gene: {type: File, outputSource: rsem/gene_out}
  RNASeQC_Metrics: {type: File, outputSource: rna_seqc/Metrics}
  RNASeQC_counts: {type: File, outputSource: supplemental/RNASeQC_counts} # contains gene tpm, gene read, and exon counts
  kallisto_Abundance: {type: File, outputSource: kallisto/abundance_out}
  ```

# D3b annoFuse Workflow

## Usage

### Inputs

```yaml
inputs:
  sample_name: {type: string, doc: "Sample name used for file base name of all outputs"}
  FusionGenome: {type: File, doc: "GRCh38_v27_CTAT_lib_Feb092018.plug-n-play.tar.gz", sbg:suggestedValue: {class: 'File', path: '5d8bb21fe4b0950c4028f854', name: 'GRCh38_v27_CTAT_lib_Feb092018.plug-n-play.tar.gz'}}
  genome_untar_path: {type: ['null', string], doc: "This is what the path will be when genome_tar is unpackaged", default: "GRCh38_v27_CTAT_lib_Feb092018/ctat_genome_lib_build_dir"}
  arriba_output_file: {type: File, doc: "Output from arriba, usually extension arriba.fusions.tsv"}
  col_num: {type: ['null', int], doc: "column number in file of fusion name", default: 25}
  star_fusion_output_file: {type: File, doc: "Output from arriba, usually extension STAR.fusion_predictions.abridged.coding_effect.tsv"}
  output_basename: string # Final file output base name
```

### Run
In this workflow annoFuse performs standardization of StarFusion and arriba output files to retain information regarding fused genes,breakpoints, reading frame information as well as annotation from FusionAnnotator, output format description [here](https://github.com/d3b-center/annoFuse/wiki#1-standardize-calls-from-fusion-callers-to-retain-information-regarding-fused-genesbreakpoints-reading-frame-information-as-well-as-annotation-from-fusionannotator)
. Basic artifact filtering to remove fusions among gene paralogs, conjoined genes and fused genes found in normal samples is also performed by filtering fusions annotated by [FusionAnnotator](https://github.com/d3b-center/FusionAnnotator) with "GTEx_Recurrent|DGD_PARALOGS|Normal|BodyMap|ConjoinG".
 Each fusion call needs atleast 1 junction reads support to be retained as true call.
 Additionally, if a fusion call has large number of spanning fragment reads compared to junction reads (spanning fragment minus junction read greater than ten), we remove these calls as potential false positives.
 Please refer to [annoFuse](https://github.com/d3b-center/annoFuse) R package for additional applications like expression based filtering and putative oncogene annotations.

1) Outputs from the arriba and STAR Fusion runs are required ahead of time (main RNAseq worflow output)
2) `FusionGenome` shoould match what was used to run STAR Fusion

### Outputs

```yaml
outputs:
  annofuse_filtered_fusions_tsv: {type: File, outputSource: annoFuse_filter/filtered_fusions_tsv, doc: "Filtred output of formatted and annotated Star Fusion and arriba results"}
```
