# Differential Expression

## What is the common outcome of an RNA-Seq analysis?
The goal of most RNA-Seq analyses is to find genes or transcripts that change across experimental conditions.
This change is called **differential expression**. By finding these genes and transcripts, we can infer the functional characteristics of the different conditions.

## Why using RNA in differential expression?
Unlike DNA, which is static, the mRNA abundances change over time.
You will need to ensure not only that you observed a change but that this change is correlated with the experimental conditions.
Typically this is achieved by measuring the same state multiple times.

## What is a "spike-in" control?
The goal of the spike-in control is to determine how well we can measure and reproduce data with known (expected) properties.
A common product called the "ERCC ExFold RNA Spike-In Control Mix" can be added in different mixtures.
This spike-in consists of 92 transcripts that are present in known
concentrations across a wide abundance range (from very few copies to many copies).

---

## Prepare the data

### Download

```sh
mkdir -p ~/workdir/diff_exp && cd ~/workdir/diff_exp/
wget -c https://0x0.st/zK57.gz -O ref.tar.gz
tar xvzf ref.tar.gz
wget -c https://raw.githubusercontent.com/mr-eyes/nu-ngs01/master/Day-6/deseq1.r
wget -c https://raw.githubusercontent.com/mr-eyes/nu-ngs01/master/Day-6/draw-heatmap.r

# Extract sample_data

```

### Does this data have "spike-in" control?
Yes there are two mixes: ERCC Mix 1 and ERCC Mix2. The spike-in consists of 92 transcripts that are present in known concentrations across a wide abundance range (from very few copies to many copies).
[More info](http://tools.thermofisher.com/content/sfs/manuals/cms_086340.pdf)

### Description

The data consists of two commercially available RNA samples:

1. Universal Human Reference **(UHR)** is total RNA isolated from a diverse set of 10 cancer cell lines. [more info](https://www.chem-agilent.com/pdf/strata/740000.pdf)
2. Human Brain Reference **(HBR)** is total RNA isolated from the brains of 23 Caucasians, male and female, of varying age but mostly 60-80 years old. [more info](https://assets.thermofisher.com/TFS-Assets/LSG/manuals/sp_6052.pdf)


#### The data was produced in three replicates for each condition.

1. UHR + ERCC Mix1, Replicate 1, **HBR_1**
2. UHR + ERCC Mix1, Replicate 2, **HBR_2**
3. UHR + ERCC Mix1, Replicate 3, **HBR_3**
4. HBR + ERCC Mix2, Replicate 1, **UHR_1**
5. HBR + ERCC Mix2, Replicate 2, **UHR_2**
6. HBR + ERCC Mix2, Replicate 3, **UHR_3**

`tree data` to see the folders structure.

---

## Setup enviornemnt

```bash
conda activate ngs1
# conda install kallisto
# conda install samtools

# Install subread, we will use featureCount : a software program developed for counting reads to genomic features such as genes, exons, promoters and genomic bins.
conda install subread

# install r and dependicies
conda install r
conda install -y bioconductor-deseq r-gplots

```


---

## Analyzing control samples

### Alignment: Hisat2

#### Step 1 (Indexing)

```bash
REF_ERCC=./ref/ERCC92.fa
INDEX_ERCC=./ref/ERCC92

hisat2-build $REF_ERCC $INDEX_ERCC

```

#### Step 2 (Alignment)

```bash
INDEX=~/workdir/diff_exp/ref/ERCC92
RUNLOG=runlog.txt
READS_DIR=~/workdir/sample_data/
mkdir bam

for SAMPLE in HBR UHR;
do
    for REPLICATE in 1 2 3;
    do
        R1=$READS_DIR/${SAMPLE}_Rep${REPLICATE}*read1.fastq.gz
        R2=$READS_DIR/${SAMPLE}_Rep${REPLICATE}*read2.fastq.gz
        BAM=bam/${SAMPLE}_${REPLICATE}.bam

        hisat2 $INDEX -1 $R1 -2 $R2 | samtools sort > $BAM
        samtools index $BAM
    done
done
```

> You can visualize BAM files using the [Integrative Genomics Viewer (IGV)](https://software.broadinstitute.org/software/igv/download)


#### Step 3 (Quantification)

```bash
GTF=~/workdir/diff_exp/ref/ERCC92.gtf

# Generate the counts.
featureCounts -a $GTF -g gene_name -o counts.txt  bam/HBR*.bam  bam/UHR*.bam

# Simplify the file to keep only the count columns.
cat counts.txt | cut -f 1,7-12 > simple_counts.txt
```

> Head

| Geneid     | bam/HBR_1.bam | bam/HBR_2.bam | bam/HBR_3.bam | bam/UHR_1.bam | bam/UHR_2.bam | bam/UHR_3.bam | 
|------------|---------------|---------------|---------------|---------------|---------------|---------------| 
| ERCC-00002 | 37892         | 47258         | 42234         | 39986         | 25978         | 33998         | 
| ERCC-00003 | 2904          | 3170          | 3038          | 3488          | 2202          | 2680          | 
| ERCC-00004 | 910           | 1078          | 996           | 9200          | 6678          | 7396          | 
| ERCC-00009 | 638           | 778           | 708           | 1384          | 954           | 1108          | 
| ERCC-00012 | 0             | 0             | 0             | 2             | 0             | 0             | 
| ERCC-00013 | 0             | 0             | 0             | 4             | 4             | 0             | 
| ERCC-00014 | 26            | 20            | 8             | 20            | 4             | 16            | 
| ERCC-00016 | 0             | 0             | 0             | 0             | 0             | 0             | 
| ERCC-00017 | 0             | 0             | 0             | 0             | 0             | 2             | 


```bash
# Analyze the counts with DESeq1.
cat simple_counts.txt | Rscript deseq1.r 3x3 > results_deseq1.tsv
```

> Head DESeq1 Output

| id         | baseMean         | baseMeanA        | baseMeanB        | foldChange       | log2FoldChange   | pval                 | padj                 | 
|------------|------------------|------------------|------------------|------------------|------------------|----------------------|----------------------| 
| ERCC-00130 | 29681.8244237545 | 10455.9218232761 | 48907.7270242329 | 4.67751460376823 | 2.22574215774208 | 1.16729711209977e-88 | 9.10491747437823e-87 | 
| ERCC-00108 | 808.597670575459 | 264.877838024487 | 1352.31750312643 | 5.10543846632202 | 2.35203486825767 | 2.40956154792562e-62 | 9.39729003690993e-61 | 
| ERCC-00136 | 1898.3382995277  | 615.744918976546 | 3180.93168007886 | 5.16598932779828 | 2.36904466305553 | 2.80841619396469e-58 | 7.30188210430821e-57 | 
| ERCC-00116 | 952.57953992746  | 337.704944218003 | 1567.45413563692 | 4.64149004174798 | 2.21458802318734 | 1.72224091670521e-45 | 3.35836978757517e-44 | 



#### DeSEQ1 Output header description

- `id`: Gene or transcript name that the differential expression is computed for,
- `baseMean`: The average normalized value across all samples,
- `baseMeanA, baseMeanB`: The average normalized gene expression for each condition,
- `foldChange`: The ratio baseMeanB/baseMeanA ,
- `log2FoldChange`: log2 transform of foldChange . When we apply a 2-based logarithm the values
become symmetrical around 0. A log2 fold change of 1 means a doubling of the expression level, a log2 fold change of -1 shows show a halving of the expression level.
- `pval`: The probability that this effect is observed by chance,
- `padj`: The adjusted probability that this effect is observed by chance.

#### View only rows with pval < 0.05

```bash
cat results_deseq1.tsv | awk ' $8 < 0.05 { print $0 }' > filtered_results_deseq1.tsv
cat filtered_results_deseq1.tsv | Rscript draw-heatmap.r > hisat_output.pdf
```

---
---

# Differential Expression using pseudoalignment

## What is Kallisto?
Kallisto is a software package for quantifying transcript abundances.
The tool perform a pseudoalignment of reads against a transcriptome, In pseudoalignment, the program tries to identify for each read the target that it originates from but not where in the target it aligns.
This makes the algorithm much faster than a 'real' alignment algorithm.

## Automate the same expirement with Kallisto

```bash
set -euo pipefail ## stop execution on errors, https://explainshell.com/explain?cmd=set+-euxo%20pipefail

# Collect program output here.
RUNLOG=runlog.log

echo "Run by `whoami` on `date`" > $RUNLOG # write log while running.

READS=~/workdir/sample_data/
REF_ERCC=~/workdir/diff_exp/ref/ERCC92.fa  # Reference
INDEX_ERCC=~/workdir/diff_exp/ref/ERCC92.idx # Index_File Name 


# Build the index if necessary.
if [ ! -f $INDEX_ERCC ] # Check if the file data/refs/ERCC92.idx exists
then
    echo "*** Building kallisto index: $INDEX_ERCC"
    kallisto index -i $INDEX_ERCC  $REF_ERCC 2>> $RUNLOG
fi

# Two output directories for control and brain samples.
DIR_ERCC=ercc_kallisto
mkdir -p $DIR_ERCC

for SAMPLE in HBR UHR;
do
    for REPLICATE in 1 2 3;
    do
        # Build the name of the files (Paired End).
        R1=$READS/${SAMPLE}_Rep${REPLICATE}*read1.fastq.gz
        R2=$READS/${SAMPLE}_Rep${REPLICATE}*read2.fastq.gz

        echo "*** Running kallisto on ${SAMPLE}_${REPLICATE} vs $INDEX_ERCC"
        OUT=$DIR_ERCC/${SAMPLE}_${REPLICATE}
        kallisto quant -i $INDEX_ERCC -o $OUT -b 100 $R1 $R2 2>> $RUNLOG

    done
done

echo "*** Created counts for ERCC control samples."
paste ${DIR_ERCC}/H*/abundance.tsv ${DIR_ERCC}/U*/abundance.tsv | cut -f 1,4,9,14,19,24,29  > ercc_kallisto_counts.tsv


echo "*** Running the DESeq1 and producing the final result: ercc_kallisto_deseq1.tsv"
cat ercc_kallisto_counts.tsv | Rscript deseq1.r 3x3 > ercc_kallisto_deseq1.tsv  2>> $RUNLOG

# Filter pval < 0.05
cat ercc_kallisto_deseq1.tsv | awk ' $8 < 0.05 { print $0 }' > filtered_kallisto_deseq1.tsv

```
> Head kallisto DeSeq1 output

| id         | baseMean         | baseMeanA        | baseMeanB        | foldChange       | log2FoldChange   | pval                 | padj                 | 
|------------|------------------|------------------|------------------|------------------|------------------|----------------------|----------------------| 
| ERCC-00130 | 14840.7854784434 | 5227.93407242622 | 24453.6368844606 | 4.67749526786055 | 2.22573619391768 | 8.73963105304599e-87 | 6.81691222137587e-85 | 
| ERCC-00108 | 404.298508194945 | 132.438264826674 | 676.158751563215 | 5.10546368489592 | 2.35204199450587 | 9.72746450192469e-59 | 3.79371115575063e-57 | 
| ERCC-00136 | 949.168332376237 | 307.870824713046 | 1590.46584003943 | 5.16601675888527 | 2.36905232365751 | 2.5211305539385e-57  | 6.55493944024009e-56 | 
| ERCC-00116 | 476.289329027567 | 168.851590236675 | 783.727067818459 | 4.64151428316386 | 2.21459555802611 | 1.93683785478913e-44 | 3.77683381683879e-43 |


---

## Visualize


```bash

cat filtered_kallisto_deseq1.tsv | Rscript draw-heatmap.r > kallisto_output.pdf

```









