# RNA-seq Differential Gene Expression Workflow  

### Author: Ziwen Xie  

## Abstract:  

This article details the entire process of RNA-seq data from download to analysis, including installing miniconda, creating a virtual environment, downloading and installing bioinformatics software, downloading data, data quality checking, raw data pruning, alignment, and calculating RNA expression. The workflow consists of six main steps:  
1. Install Miniconda  
2. Install necessary Conda packages
3. Create a virtual environment
4. Analyze sequence quality with FastQC  
5. Raw data pruning with Trim_Galore  
6. Calculate the amount of RNA expression

## Introduction:  

RNA-Seq is a high-throughput sequencing technology used to study RNA expression in biological systems. Compared to traditional microarray techniques, RNA-Seq offers higher sensitivity and a broader dynamic range, making it widely applicable for analyzing gene expression, transcript structures, RNA editing, and alternative splicing in various biological processes.  

## Dataset and Methods:  

The dataset used includes a sample file from the EBI database and SortMeRNA's rRNA database.  

### Methodology (recommended with root privileges):  

1. **Clone the repository locally**  
```shell
git clone https://github.com/Shyreak/RNA-seq
```

2. **Run the following script to install Miniconda**  
```sh
 wget -c  https://mirrors.tuna.tsinghua.edu.cn/anaconda/miniconda/Miniconda3-latest-Linux-x86_64.sh
```
Or download it from the official website as you like

```sh
 wget -c https://repo.continuum.io/miniconda/Miniconda3-latest-Linux-x86_64.sh
```

For those in China, Switching to Tsinghua mirror source is highly recommended.
```sh
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main/
conda config --set show_channel_urls yes
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```
3. **Create a virtual environment**
conda create -n rna python=3 

conda info --envs 

source activate rna 
conda activate rna 

 source deactivate 

conda config --set auto_activate_base false

4. **Install necessary Conda packages**  
```shell
conda search sra 

conda install -y sra-tools
conda install -y trimmomatic
conda install -y cutadapt multiqc 
conda install -y trim-galore
conda install -y star hisat2 bowtie2
conda install -y subread tophat htseq bedtools deeptools
conda install -y salmon
```

5. **Download data**  
Create a new document named SRR_Acc_List.txt and save the SRR number in the document, with one number occupying one line.
Set the directory first.
```shell
wkd=/lab/project
cd /lab/project

cat SRR_Acc_List.txt | while read id; do (prefetch  ${id} &);done
```

6. **Test data quality**  
Command format:  
```shell
cd $wkd/

fastq-dump *.sra

fastqc *.fastq

multiqc *.zip

```  
Output:  
```
── results/2_trimmed_output/
     └── sample_trimmed.fq                 <- Trimmed sequencing file (.fastq)
     └── sample_trimmed.html               <- HTML quality report
     └── sample_trimmed.zip                <- Quality report data
     └── sample.fastq.trimming_report.txt  <- Cutadapt trimming report
```

7. **Run the fifth script to build index database and remove rRNA sequences with SortMeRNA**  
**Note: Requires decompressed input files**  
```sh
# Install environment
conda install -c bioconda sortmerna=2.1b
source ~/.bashrc

# Decompress data file
gunzip results/2_trimmed_output/sample_trimmed.fq.gz 

# Build index database
wget -P sortmerna_db https://github.com/biocore/sortmerna/archive/2.1b.zip
unzip sortmerna_db/2.1b.zip -d sortmerna_db
mv sortmerna_db/sortmerna-2.1b/rRNA_databases/ sortmerna_db/
rm sortmerna_db/2.1b.zip
rm -r sortmerna_db/sortmerna-2.1b

# Configure database references
sortmernaREF=sortmerna_db/rRNA_databases/silva-arc-16s-id95.fasta,sortmerna_db/index/silva-arc-16s-id95:\
sortmerna_db/rRNA_databases/silva-arc-23s-id98.fasta,sortmerna_db/index/silva-arc-23s-id98:\
sortmerna_db/rRNA_databases/silva-bac-16s-id90.fasta,sortmerna_db/index/silva-bac-16s-id95:\
sortmerna_db/rRNA_databases/silva-bac-23s-id98.fasta,sortmerna_db/index/silva-bac-23s-id98:\
sortmerna_db/rRNA_databases/silva-euk-18s-id95.fasta,sortmerna_db/index/silva-euk-18s-id95:\
sortmerna_db/rRNA_databases/silva-euk-28s-id98.fasta,sortmerna_db/index/silva-euk-28s-id98

# Run indexing command
indexdb_rna --ref $sortmernaREF

# Execute script (default input: results/2_trimmed_output/sample_trimmed.fq)
bash 5_SortMeRNA.sh $sortmernaREF
```  
Output:  
```
── results/4_aligned_sequences/
    └── aligned_bam/sampleAligned.sortedByCoord.out.bam   <- Sorted BAM alignment file
    └── aligned_logs/sampleLog.final.out                  <- STAR alignment rate log
    └── aligned_logs/sampleLog.out                        <- STAR alignment process log
```

## Results:  

```
── new_workflow/
  │  
  │   └── input/                    <- Input RNAseq data
  │  
  │   └── output/                   <- Processed data outputs
  │       ├── 1_initial_qc/         <- Initial quality control files
  │       ├── 2_trimmed_output/     <- Trimmed sequence outputs
  │       ├── 3_rRNA/               <- rRNA processing results
  │           ├── aligned/          <- rRNA-contaminated sequences
  │           ├── filtered/         <- rRNA-free sequences
  │           ├── logs/             <- SortMeRNA execution logs
  │   └── sortmerna_db/             <- rRNA databases for SortMeRNA
  │       ├── index/                <- Indexed rRNA sequences
  │       ├── rRNA_databases/       <- rRNA sequences from bacteria, archea, eukaryotes
  │  
  │   └── star_index/               <- Indexed genome files from STAR 
```

* Successfully generated FastQC quality analysis reports  
* Obtained quality-trimmed sample files with corresponding quality reports  
* Generated Trim_Galore trimming reports  
* Produced SortMeRNA removal reports  

## Discussion:  

**Key project aspects:**  
Utilized FastQC, Trim_Galore for processing target genes, generating:  
- FastQC quality analysis reports  
- Quality reports for low-quality sequence removal  
- Trim_Galore trimming reports  

**Limitations:**  
- Limited processing methods for target gene samples  
- Analysis reports could be more comprehensive  
- Workflow lacks differential expression analysis steps  

**Future improvements:**  
- Incorporate alignment (STAR/HISAT2) and quantification (featureCounts) steps  
- Add differential expression analysis (DESeq2/edgeR)  
- Include visualization components (PCA, volcano plots)  
- Implement multi-sample processing capability  

## References:  

1. Andrews S. (2010). FastQC: a quality control tool for high throughput sequence data. Available online at: http://www.bioinformatics.babraham.ac.uk/projects/fastqc  
2. Martin, Marcel. Cutadapt removes adapter sequences from high-throughput sequencing reads. EMBnet.journal, [S.l.], v. 17, n. 1, p. pp. 10-12, may. 2011. ISSN 2226-6089. Available at: http://journal.embnet.org/index.php/embnetjournal/article/view/200. doi:http://dx.doi.org/10.14806/ej.17.1.200.  
3. Kopylova E., Noé L. and Touzet H., "SortMeRNA: Fast and accurate filtering of ribosomal RNAs in metatranscriptomic data", Bioinformatics (2012), doi: 10.1093/bioinformatics/bts611  
4. Dobin A, Davis CA, Schlesinger F, et al. STAR: ultrafast universal RNA-seq aligner. Bioinformatics. 2013;29(1):15-21. doi:10.1093/bioinformatics/bts635.  
5. Lassmann et al. (2010) "SAMStat: monitoring biases in next generation sequencing data." Bioinformatics doi:10.1093/bioinformatics/btq614 [PMID: 21088025]  

## Appendix:  

Core scripts and Dockerfile available on GitHub.
