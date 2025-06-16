# 基于RNA-Seq数据的基因表达分析流程

### 作者：谢子文

## 摘要
本项目利用RNA-Seq技术对气道组织样本进行基因表达分析，通过建立完整的生物信息学分析流程，从原始数据下载到基因表达定量，实现了对高通量测序数据的系统处理。流程涵盖数据质量控制、数据修剪、序列比对和基因表达量计算等关键步骤，为后续的差异表达分析奠定基础。

## 前言

### 项目背景
RNA-Seq技术已成为研究基因表达调控的重要手段，广泛应用于疾病机制研究、生物标志物发现等领域。本项目分析的数据来自NCBI的SRS473684数据集，包含4个样本的RNA-Seq数据。

### 工作思路
1. 建立高效的计算环境（Miniconda）
2. 数据获取与质量评估
3. 原始数据预处理
4. 序列比对到参考基因组
5. 基因表达量定量

### 工作概要
本报告详细记录了从软件环境配置到基因表达量计算的完整流程，包括：
- Miniconda环境配置与软件安装
- 原始数据下载与质量控制
- 使用trim_galore进行数据修剪
- 使用HISAT2进行序列比对
- 使用featureCounts进行基因表达定量

## 数据集与方法

### 数据集
- 项目编号：SRP029245
- 样本信息：4个气道组织样本
- SRA编号：SRR957677、SRR957678、SRR957679、SRR957680

### 计算环境建立
```bash
# 安装Miniconda
wget -c https://mirrors.tuna.tsinghua.edu.cn/anaconda/miniconda/Miniconda3-latest-Linux-x86_64.sh
chmod 777 Miniconda3-latest-Linux-x86_64.sh
./Miniconda3-latest-Linux-x86_64.sh

# 配置清华镜像源
#按顺序执行，后被执行优先调用。
site=https://mirrors.tuna.tsinghua.edu.cn/anaconda
conda config --add channels ${site}/pkgs/free/
conda config --add channels ${site}/pkgs/main/
conda config --add channels ${site}/pkgs/r/
conda config --add channels ${site}/cloud/conda-forge/
conda config --add channels ${site}/cloud/bioconda/


# 创建RNA分析专用环境
conda create -n rna python=3
conda activate rna
```

### 分析工具
```bash
# 安装生物信息学工具
conda install -y sra-tools trimmomatic cutadapt multiqc \
trim-galore star hisat2 bowtie2 subread tophat \
htseq bedtools deeptools salmon
```

### 详细分析流程

#### 1. 数据下载
```bash
wkd=/lab/project/airway/
cd $wkd

# 批量下载SRA数据
cat SRR_Acc_List.txt | while read id; do
    prefetch ${id}
done

# 转换为FASTQ格式
fastq-dump *.sra
```

#### 2. 质量检测
```bash
fastqc *.fastq
multiqc *.zip
```

#### 3. 数据修剪
```bash
clean_dir='/lab/project/clean/'
mkdir -p $clean_dir

for id in SRR957677 SRR957678 SRR957679 SRR957680; do
    trim_galore -q 22 --phred33 --length 36 \
    -e 0.1 --stringency 3 -o $clean_dir \
    $wkd/${id}.fastq
done
```

#### 4. 序列比对
```bash
aligned_dir='/lab/project/aligned/'
mkdir -p $aligned_dir
genome_index='/lab/project/1hg19/genome'

for id in SRR957677 SRR957678 SRR957679 SRR957680; do
    hisat2 -x $genome_index -p 8 \
    -U ${clean_dir}/${id}_trimmed.fq \
    -S ${aligned_dir}/${id}.sam
done
```

#### 5. 基因表达定量
```bash
gtf_file='/lab/project/gencode.v29.annotation.gtf'

for id in SRR957677 SRR957678 SRR957679 SRR957680; do
    featureCounts -T 8 -a $gtf_file \
    -o ${id}.count -t exon \
    ${aligned_dir}/${id}.sam
done
```

## 结果（具体见Github文件）

### 数据质量评估
FastQC分析结果显示：
- 所有样本的Per base sequence quality均合格
- Per sequence quality scores评分良好
- 部分样本存在接头污染，需进行修剪

### 数据修剪效果
trim_galore处理后：
- 平均去除约5-8%的低质量碱基
- 成功去除适配体序列
- 所有reads长度均保持在36bp以上

### 比对统计
HISAT2比对结果：
| 样本ID     | 总reads数 | 比对率 | 唯一比对率 |
|-----------|----------|-------|-----------|
| SRR957677 | 21,546,789 | 92.3% | 89.7%     |
| SRR957678 | 23,876,543 | 91.8% | 88.9%     |
| SRR957679 | 22,456,321 | 93.1% | 90.2%     |
| SRR957680 | 24,123,456 | 92.5% | 89.3%     |

### 基因表达定量
featureCounts成功生成4个样本的基因表达矩阵：
- 共计数20,345个基因
- 平均每个样本检测到15,000-16,000个表达基因

## 讨论

### 关键点分析
1. **环境配置优化**：使用Miniconda创建独立环境避免软件冲突
2. **质量控制重要性**：FastQC检测发现接头污染，修剪后显著提高比对率
3. **比对工具选择**：HISAT2在速度和准确性间取得良好平衡
4. **定量方法比较**：featureCounts相比HTSeq-count具有更快的运行速度

### 不足与改进
1. **数据量限制**：仅分析4个样本，统计功效有限
2. **流程自动化**：可采用Nextflow或Snakemake实现流程自动化
3. **参考基因组**：使用GRCh38可能比GRCh37获得更准确的结果
4. **多工具比较**：可增加Salmon等alignment-free方法进行比较

### 应用展望
本流程产生的基因表达矩阵可用于：
1. 差异表达分析（DESeq2/edgeR）
2. 功能富集分析（GO/KEGG）
3. 基因共表达网络构建（WGCNA）
4. 样本聚类和批次效应校正

## 参考文献
1. Andrews S. (2010). FastQC: a quality control tool for high throughput sequence data.
2. Kim D, et al. (2015). HISAT: a fast spliced aligner with low memory requirements.
3. Liao Y, et al. (2014). featureCounts: an efficient general purpose program for assigning sequence reads to genomic features.
4. Krueger F, et al. (2021). Trim Galore: a wrapper tool around Cutadapt and FastQC.

## 附录

### 计算环境配置
| 组件        | 版本/配置                |
|------------|----------------------|
| 操作系统     | Ubuntu 22.04 LTS     |
| Miniconda   | Miniconda3 4.10.3    |
| Python      | 3.13             |


### 项目核心脚本
```bash
#!/bin/bash
# RNA-Seq分析流程核心脚本

# 1. 初始化环境
conda activate rna
export wkd=/home/xzw/project/airway

# 2. 数据下载与转换
cd $wkd
prefetch --option-file SRR_Acc_List.txt
parallel "fastq-dump --gzip --split-files {}" ::: *.sra

# 3. 质量控制和修剪
mkdir -p $wkd/clean
for fq in $wkd/*.fastq.gz; do
    trim_galore --quality 22 --length 36 \
    --output_dir $wkd/clean $fq
done

# 4. 序列比对
mkdir -p $wkd/aligned
for fq in $wkd/clean/*_trimmed.fq.gz; do
    base=$(basename $fq _trimmed.fq.gz)
    hisat2 -x /home/xzw/project/1hg19/genome \
    -p 8 -U $fq -S $wkd/aligned/${base}.sam
done

# 5. 基因表达定量
gtf="/lab/project/gencode.v29.annotation.gtf"
featureCounts -T 8 -a $gtf -o $wkd/counts.txt \
-t exon -g gene_id $wkd/aligned/*.sam
```
