# scReadSim for 10x scRNA-seq (exon-only)


This tutorial's main steps and corresponding estimated time usage are as follows (tested on a server with the 256x Intel Xeon Phi CPU 7210 at 1.30 GHz):

<!-- - [Step 1: Import packages and data files](#step-1-import-packages-and-data-files): < 1 min
- [Step 2: Generate features](#step-2-generate-features): < 1 min
- [Step 3: Generate real count matrices](#step-3-generate-real-count-matrices): ~ 3 mins
- [Step 4: Simulate synthetic count matrix](#step-4-simulate-synthetic-count-matrix): ~ 8 mins
- [Step 5: Output synthetic read](#step-5-output-synthetic-read): ~ 3 mins -->
- **Step 1: Import packages and data files**: < 1 min
- **Step 2: Generate features**: < 1 min
- **Step 3: Generate real count matrices**: ~ 3 mins
- **Step 4: Simulate synthetic count matrix**: ~ 6 mins
- **Step 5: Output synthetic read**: ~ 10 mins

By default, this tutorial uses Python (Python >= 3.8). However, we also include code chunks using bash commands to preprocess necessary files. To avoid users' confusion, bash commands start with a symbol **$**. We also indicate when a following code chunk is using bash commands. 


## Required softwares for scReadSim
scReadSim requires users to pre-install the following softwares:
- [samtools >= 1.12](http://www.htslib.org/)
- [bedtools >= 2.29.1](https://bedtools.readthedocs.io/en/latest/)
- [seqtk >= 1.3](https://github.com/lh3/seqtk)
- [fgbio >= 2.0.1](https://github.com/fulcrumgenomics/fgbio)

Depending on users' choices, the following softwares are optional:
- [bowtie2](http://bowtie-bio.sourceforge.net/bowtie2/index.shtml)


## Pre-process BAM file before scReadSim
**Note**: This tutorial does not need this pre-process step since the processed BAM file is provided by the scReadSim package (see below **Step 1: Import packages and data files**).

Input BAM file for scReadSim needs pre-processing to add the cell barcode in front of the read name. For example, in 10x sequencing data, cell barcode `TGGACCGGTTCACCCA-1` is stored in the field `CB:Z:TGGACCGGTTCACCCA-1`. 

The following code chunk (**bash commands**) outputs a read record from the original BAM file.

```{code-block} console
$ samtools view unprocess.bam | head -n 1
A00836:472:HTNW5DMXX:1:1372:16260:18129      83      chr1    4194410 60      50M     =       4193976 -484    TGCCTTGCTACAGCAGCTCAGGAAATGTCTTTGTGCCCACAGTCTGTGGT   :FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF      NM:i:0  MD:Z:50 AS:i:50 XS:i:0  CR:Z:TCCGGGACAGCTAACA   CY:Z:FFFFFFFFFFFFFFF:   CB:Z:TGGACCGGTTCACCCA-1 BC:Z:AAACTCAT        QT:Z::FFFFFFF   RG:Z:e18_mouse_brain_fresh_5k:MissingLibrary:1:HTNW5DMXX:1
```

The following code chunk (**bash commands**) adds the cell barcodes in front of the read names.

```{code-block} console
$ # extract the header file
$ mkdir tmp
$ samtools view unprocess.bam -H > tmp/unprocess.header.sam

$ # create a bam file with the barcode embedded into the read name
$ time(cat <( cat tmp/unprocess.header.sam ) \
 <( samtools view unprocess.bam | awk '{for (i=12; i<=NF; ++i) { if ($i ~ "^CB:Z:"){ td[substr($i,1,2)] = substr($i,6,length($i)-5); } }; printf "%s:%s\n", td["CB"], $0 }' ) \
 | samtools view -bS - > processed.bam) 
$ rm -dr tmp

$ samtools view processed.bam | head -n 1
TGGACCGGTTCACCCA-1:A00836:472:HTNW5DMXX:1:1372:16260:18129      83      chr1    4194410 60      50M     =       4193976 -484    TGCCTTGCTACAGCAGCTCAGGAAATGTCTTTGTGCCCACAGTCTGTGGT   :FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF      NM:i:0  MD:Z:50 AS:i:50 XS:i:0  CR:Z:TCCGGGACAGCTAACA   CY:Z:FFFFFFFFFFFFFFF:   CB:Z:TGGACCGGTTCACCCA-1 BC:Z:AAACTCAT        QT:Z::FFFFFFF   RG:Z:e18_mouse_brain_fresh_5k:MissingLibrary:1:HTNW5DMXX:1
```




## Download reference genome for test example
The example deploys scReadSim on the [10x single cell RNA-seq](https://www.10xgenomics.com/resources/datasets/fresh-embryonic-e-18-mouse-brain-5-k-1-standard-2-0-0) dataset. For user convienience, we prepared the indexed reference genome files (by bowtie2), which can be downloaded using the following bash commands:
- GENCODE reference genome FASTA file and index file(indexed by bowtie2): *reference.genome.chr1.tar.gz*
- GENCODE genome annotation gtf file: *gencode.vM10.annotation.gtf*

**Note**: users may need to edit the code by using their own path. The following code chunk is using **bash commands**.


```{code-block} console
$ mkdir /home/users/example/refgenome_dir # may use users' own path
$ cd /home/users/example/refgenome_dir
$ wget http://compbio10data.stat.ucla.edu/repository/gayan/Projects/scReadSim/reference.genome.chr1.tar.gz # 292 MB
$ wget http://compbio10data.stat.ucla.edu/repository/gayan/Projects/scReadSim/gencode.vM10.annotation.gtf # 765 MB
$ tar -xf reference.genome.chr1.tar.gz
```

## Download collapsed transcriptome for test example
In order to generate scRNA-seq synthetic reads from exon-only regions, scReadSim requires a collapsed transcriptome FASTA file where all transcripts for a gene are merged into a single one (similar to collapsed transcriptome used in [UMI-tools](https://umi-tools.readthedocs.io/en/latest/Single_cell_tutorial.html#generating-merge-transcriptome-annotations) and superTranscript concept proposed by [Davidison et al.](https://genomebiology.biomedcentral.com/articles/10.1186/s13059-017-1284-1)). For user convienience, we prepared the collapsed transcriptome file (based on [GENCODE mouse GRCm38](https://www.gencodegenes.org/mouse/release_M10.html) reference genome file and annotation gtf file), which can be downloaded directly using the following bash commands:
- Mouse GRCm38 chr1 collapsed transcriptome file: *gencode.vM10.chr1.merged.fa*

**Note**: users may need to edit the code by using their own path. The following code chunk is using **bash commands**.


```{code-block} console
$ cd /home/users/example/refgenome_dir
$ wget http://compbio10data.stat.ucla.edu/repository/gayan/Projects/scReadSim/gencode.vM10.chr1.merged.fa # 9.2 MB
```

### (Optional) Generate users' own collapsed transcriptome

To generate the above collapsed transcriptome FASTA file, we used `cgat` [package](https://github.com/cgat-developers/cgat-apps) and `gffread` [software](http://ccb.jhu.edu/software/stringtie/gff.shtml#gffread): 

```{code-block} console
$ # Step 1: merge all transcripts for one gene
$ grep transcript_id gencode.vM10.merged.gtf \
        | cgat gtf2gtf --method=merge-exons \ 
                | cgat gtf2gtf --method=set-transcript-to-gene -S gencode.vM10.merged.gtf

$ # Step 2: Generate fa based on merged annotation gtf
$ gffread -w gencode.vM10.chr1.merged.fa \
        -g chr1.fa \
        -E \
        gencode.vM10.merged.gtf
```

## Step 1: Import packages and data files

Import modules.

```{code-block} python3
import sys, os
import scReadSim.Utility as Utility
import scReadSim.GenerateSyntheticCount as GenerateSyntheticCount
import scReadSim.scRNA_GenerateBAM as scRNA_GenerateBAM
import pkg_resources
```

The real BAM file and other input files are listed and can be accessed by simply loading the code chunk below:
-  BAM file: *10X_RNA_chr1_3073253_4526737.bam*
-  cell barcode file: *barcodes.tsv*
-  cell condition file: *10X_RNA_conditionlabel.txt*
-  chromosome size file: *mm10.chrom.sizes*


```{code-block} python3
INPUT_cells_barcode_file = pkg_resources.resource_filename("scReadSim", 'data/barcodes.tsv') 

filename = "10X_RNA_chr1_3073253_4526737"
INPUT_bamfile = pkg_resources.resource_filename("scReadSim", 'data/%s.bam' % filename)
INPUT_genome_size_file = pkg_resources.resource_filename("scReadSim", 'data/mm10.chrom.sizes')
```


## Step 2: Generate features
To pre-process real scRNA-seq data for training, scReadSim requires a BAM file (containing scRNA-seq reads in cells) and a gene annotation file (in GTF format). Based on the gene coordinates in the annotation file, scReadSim segregates the reference genome into two sets of features: genes and inter-genes.


### Specify output directory
**Note**: users may need to edit the code by using their own path.

```{code-block} python3
outdirectory = "/home/users/example/outputs" # may use user's own path
os.mkdir(outdirectory)
```


### Specify pre-installed software paths
**Note**: users may need to edit the code by using their own path.


```{code-block} python3
samtools_directory="/home/users/Tools/samtools" 
macs3_directory="/home/users/Tools/MACS3/bin"
bedtools_directory="/home/users/Tools/bedtools/bedtools2/bin"
seqtk_directory="/home/users/Tools/seqtk"
fgbio_jarfile="/home/users/Tools/fgbio/target/scala-2.13/fgbio-2.0.1-e884860-SNAPSHOT.jar"
```

### Prepare features
Given the input BAM file and gene annotation file, scReadSim prepares the bed files for features using function `scRNA_CreateFeatureSets`. Here, for a quick implementation in this demo, we have prepared two smaller feature files (embedded within the package). Users only need to implement the following code chunk to import them into the python environment. To generate features using scReadSim, please refer to section "Prepare features" in tutorial **scReadSim for 10x scRNA-seq**.


**Note**: users may need to edit the code by using their own path.

```{code-block} python3
# Specify the absolute path to gene annotation file
INPUT_genome_annotation = "/home/users/example/refgenome_dir/gencode.vM10.annotation.gtf" # may use user's own path

# Load feature files
gene_bedfile = pkg_resources.resource_filename("scReadSim", 'data/scReadSim.head10.Gene.bed') 
intergene_bedfile = pkg_resources.resource_filename("scReadSim", 'data/scReadSim.head10.InterGene.bed')  
```


## Step 3: Generate real count matrices

Based on the feature sets output in **Step 2**, scReasSim constructs the UMI count matrices for genes and inter-genes through function `Utility.scRNA_bam2countmat_paral`. This function needs user to specify

- `cells_barcode_file`: Cell barcode file corresponding to the input BAM file.
- `bed_file`: Features bed file to generate the count matrix.
- `INPUT_bamfile`: Input BAM file for anlaysis.
- `outdirectory`: Specify the output directory of the count matrix file.
- `count_mat_filename`: Specify the base name of output read (or UMI) count matrix.
- `UMI_modeling`: (Optional, default: True) Specify whether scReadSim should model UMI count of the input BAM file.
- `UMI_tag`: (Optional, default: 'UB:Z') If UMI_modeling is set to True, specify the UMI tag of input BAM file, default value 'UB:Z' is the UMI tag for 10x scRNA-seq.
- `n_cores`: (Optional, default: '1') Specify the number of cores for parallel computing when generating count matrix.

For the user specified `count_mat_filename`, scReadSim will generate a count matrix named *`count_mat_filename`.txt* to directory `outdirectory`.



```{code-block} python3
# Specify the output count matrices' prenames
UMI_gene_count_mat_filename = "%s.gene.countmatrix" % filename
UMI_intergene_count_mat_filename = "%s.intergene.countmatrix" % filename

# Construct count matrix for genes
Utility.scRNA_bam2countmat_paral(cells_barcode_file=INPUT_cells_barcode_file, bed_file=gene_bedfile, INPUT_bamfile=INPUT_bamfile, outdirectory=outdirectory, count_mat_filename=UMI_gene_count_mat_filename, UMI_modeling=True, UMI_tag = "UB:Z", n_cores=8)
# Construct count matrix for inter-genes
Utility.scRNA_bam2countmat_paral(cells_barcode_file=INPUT_cells_barcode_file, bed_file=intergene_bedfile, INPUT_bamfile=INPUT_bamfile, outdirectory=outdirectory, count_mat_filename=UMI_intergene_count_mat_filename, UMI_modeling=True, UMI_tag = "UB:Z", n_cores=8)
```

## Step 4: Synthetic count matrix simulation

### Detect doublet (optional)
Before generating synthetic count matrices, we recommend users to detect doublets/multiplets using the real count matrices generated from previous step `Utility.scRNA_bam2countmat_paral`. This step could help remove the potential artifact effects generated from the combined profiles. scReadSim implicitly implements R package [scDblFinder](https://bioconductor.org/packages/release/bioc/vignettes/scDblFinder/inst/doc/introduction.html#scdblfinder) to identify doublets/multiplets. Use function `DoubletDetection.detectDouble` to detect the doublets/multiplets with following paramters 

- `count_mat_filename`: Base name of the count matrix output by function `Utility.scATAC_bam2countmat_paral` or `Utility.scRNA_bam2countmat_paral`.
- `directory`: Path to the count matrix.
- `outdirectory`: Specify the output directory of the synthetic count matrix file.
- `omic_choice`: Specify the omic choice for doublet detection procedure: "ATAC" or "RNA".	

The doublet detection result *doublet_classification.Rdata* will be generated to path `outdirectory`.

**Note**: Although by implementing function `DoubletDetection.detectDoublet`, scReadSim implicitly helps install the R package `scDblFinder`. However, the installation of `scDblFinder` may take a while, we recommend users to pre-install it independently in R before implementing our function `DoubletDetection.detectDoublet`.

```{code-block} python3
# Import module
import scReadSim.DoubletDetection as DoubletDetection
# Detect doublets
DoubletDetection.detectDoublet(count_mat_filename=UMI_gene_count_mat_filename, directory=outdirectory, outdirectory=outdirectory, omic_choice= "RNA")
```

### Simulate

In this tutorial, scReadSim implements [scDesign2](https://github.com/JSB-UCLA/scDesign2) to generate synthetic count matrix based on the constructed count matrix from the input BAM file. Use function `GenerateSyntheticCount.scRNA_GenerateSyntheticCount` to generate synthetic count matrix with following paramters

- `count_mat_filename`: Base name of the count matrix output by function `Utility.scRNA_bam2countmat_paral`.
- `directory`: Path to the count matrix.
- `outdirectory`: Output directory of coordinate files.
- `doub_classification_label_file`: (Optional, default: 'None') Specify the absolute path to the doublet classification result `doublet_classification.Rdata` generated by function `DoubletDetection.detectDoublet`.
- `n_cell_new`: (Optional, default: None) Number of synthetic cells. If not specified, scReadSim uses the number of real cells.
- `total_count_new`: (Optional, default: None) Number of (expected) sequencing depth. If not specified, scReadSim uses the real sequencing depth.
- `celllabel_file`: (Optional, default: None) Specify the one-column text file containing the predefined cell labels. Make sure that the order of cell labels correspond to the cell barcode file. If no cell labels are specified, scReadSim performs a Louvain clustering before implementing scDesign2.
- `n_cores`: (Optional, default: '1') Specify the number of cores for parallel computing when generating count matrix.

Given the input count matrix *`count_mat_filename`.txt*, scReadSim generates the syntheitic count matrix file to `outdirectory` for following analysis:

- Synthetic count matrix: *`count_mat_filename`.scDesign2Simulated.txt*
- Synthetic cell cluster/type labels: *`count_mat_filename`.scDesign2Simulated.CellTypeLabel.txt*

Additionaly, if no `celllabel_file` is specified, scReadSim automatically performs Louvain clustering from Seurat and outputs clustering labels to `outdirectory`:
- Real cells' Louvain clustering labels: *`count_mat_filename`.LouvainClusterResults.txt*


```{code-block} python3
# Generate synthetic count matrix for gene-by-cell count matrix
GenerateSyntheticCount.scRNA_GenerateSyntheticCount(count_mat_filename=UMI_gene_count_mat_filename, directory=outdirectory, outdirectory=outdirectory)

# Specify cluster labels obtained from peak-by-cell matrix
celllabel_file = outdirectory + "/" + "10X_RNA_chr1_3073253_4526737.gene.countmatrix.LouvainClusterResults.txt"
# Generate synthetic count matrix for non-gene-by-cell count matrix
GenerateSyntheticCount.scRNA_GenerateSyntheticCount(count_mat_filename=UMI_intergene_count_mat_filename, directory=outdirectory, outdirectory=outdirectory, celllabel_file=celllabel_file)
```

## Step 5: Output synthetic read

### Generate exon-only synthetic reads for genes
Based on the synthetic count matrix, scReadSim generates synthetic reads by randomly sampling from transcriptome file input by users. First use function `scRNA_GenerateBAMCoord_ExonOnly` to create the synthetic reads and output in BED file storing the coordinates information. Function `scRNA_GenerateBAMCoord_ExonOnly` takes following input arguments:
- `bed_file`: Features' bed file to generate the synthetic reads (Generated by function `Utility.scRNA_CreateFeatureSets`).
- `UMI_count_mat_file`: The path to the **synthetic UMI count matrix** generated by `GenerateSyntheticCount.scRNA_GenerateSyntheticCount`.
- `synthetic_cell_label_file`: Synthetic cell label file generated by `scRNA_GenerateSyntheticCount`.
- `INPUT_bamfile`: Input BAM file for anlaysis.
- `isoform_annotation_file`: Path to the isoform annotation gtf file.
- `trancriptome_file`: Path to the collapsed transcriptome file.
- `outdirectory`: Specify the output directory for synthetic reads bed file.
- `read_bedfile_prename`: Specify the base name of output bed file.
- `OUTPUT_cells_barcode_file`: Specify the file name storing the synthetic cell barcodes.
- `UMI_tag`: If UMI_modeling is set to True, specify the UMI tag of input BAM file, default value 'UB:Z' is the UMI tag for 10x scRNA-seq.
- `read_len`: Specify the length of synthetic reads. Default value is 90 bp.
- `UMI_len`: Specify the length of synthetic reads. Default value is 10 bp.

This function will output a bed file *`read_bedfile_prename`.read.bed* storing the coordinates information of synthetic reads (relative to collapsed transcriptome FASTA) and its cell barcode file `OUTPUT_cells_barcode_file` in directory `outdirectory`.

**Note**: users may need to edit the code by using their own path.

```{code-block} python3
# Specify the names of synthetic count matrices (generated by GenerateSyntheticCount.scRNA_GenerateSyntheticCount)
synthetic_countmat_gene_file = UMI_gene_count_mat_filename + ".scDesign2Simulated.txt"

# Specify the base name of bed files containing synthetic reads
OUTPUT_cells_barcode_file = "synthetic_cell_barcode.txt"
gene_read_bedfile_prename = "%s.syntheticBAM.gene" % filename
synthetic_cell_label_file = UMI_gene_count_mat_filename + ".scDesign2Simulated.CellTypeLabel.txt"

# Specify collapsed transcriptome file
trancriptome_file = "/home/users/example/refgenome_dir/gencode.vM10.chr1.merged.fa"

scRNA_GenerateBAM.scRNA_GenerateBAMCoord_ExonOnly(bed_file=gene_bedfile, 
                                UMI_count_mat_file=outdirectory + "/" + synthetic_countmat_gene_file, 
                                synthetic_cell_label_file = outdirectory + "/" + synthetic_cell_label_file, 
                                INPUT_bamfile = INPUT_bamfile, 
                                isoform_annotation_file=INPUT_genome_annotation, 
                                trancriptome_file = trancriptome_file, 
                                outdirectory = outdirectory, 
                                read_bedfile_prename = gene_read_bedfile_prename, 
                                OUTPUT_cells_barcode_file=OUTPUT_cells_barcode_file)
```

Use function `scRNA_GenerateBAM.scRNA_BED2FASTQ` to convert BED file to FASTQ file. This function takes the following arguments:
- `bedtools_directory`: Path to software bedtools.
- `seqtk_directory`: Path to software seqtk.
- `referenceGenome_file`: Reference genome FASTA file that the synthteic reads should align.
- `outdirectory`: Output directory of the synthteic bed file and its corresponding cell barcodes file.
- `BED_filename_combined`: Base name of the combined bed file output by function `scRNA_CombineBED`.
- `synthetic_fastq_prename`: Specify the base name of the output FASTQ files.

This function will output paired-end reads in FASTQ files named as *`BED_filename_combined`.read1.bed2fa.sorted.fq*, *`BED_filename_combined`.read2.bed2fa.sorted,fq* to directory `outdirectory`.

**Note**: users may need to edit the code by using their own path.

```{code-block} python3
referenceGenome_name = "chr1"
referenceGenome_dir = "/home/users/example/refgenome_dir"  # may use users' own path
referenceGenome_file = "%s/%s.fa" % (referenceGenome_dir, referenceGenome_name)
synthetic_fastq_prename = BED_filename_combined_pre

# Convert bed file into FASTQ files
scRNA_GenerateBAM.scRNA_BED2FASTQ(bedtools_directory=bedtools_directory, seqtk_directory=seqtk_directory, referenceGenome_file=trancriptome_file, outdirectory=outdirectory, BED_filename_combined=gene_read_bedfile_prename, synthetic_fastq_prename = gene_read_bedfile_prename)
```

### Generate synthetic reads for intergenic regions
For intergenic regions, scReadSim directly uses default mode to sample synthetic read starting position from real BAM file. Basically, scReadSim uses function `scRNA_GenerateBAM.scRNA_GenerateBAMCoord` to generate synthetic read coordinate BED file and function `scRNA_GenerateBAM.scRNA_BED2FASTQ` to convert BED file to FASTQ file. 
. Refer to Section "Step 5: Output synthetic read" in tutorial **scReadSim for 10x scRNA-seq** for further details.

```{code-block} python3
# Specify the names of synthetic count matrices (generated by GenerateSyntheticCount.scRNA_GenerateSyntheticCount)
synthetic_countmat_intergene_file = UMI_intergene_count_mat_filename + ".scDesign2Simulated.txt"
# Specify the base name of bed files containing synthetic reads
intergene_read_bedfile_prename = "%s.syntheticBAM.intergene" % filename

# Create synthetic read coordinates for intergenes
scRNA_GenerateBAM.scRNA_GenerateBAMCoord(
        bed_file=intergene_bedfile, UMI_count_mat_file=outdirectory + "/" + synthetic_countmat_intergene_file, synthetic_cell_label_file=outdirectory + "/" + synthetic_cell_label_file, read_bedfile_prename=intergene_read_bedfile_prename, INPUT_bamfile=INPUT_bamfile, outdirectory=outdirectory, OUTPUT_cells_barcode_file=OUTPUT_cells_barcode_file, jitter_size=5, read_len=90)

# Generate FASTQ for inter-genes
scRNA_GenerateBAM.scRNA_BED2FASTQ(bedtools_directory=bedtools_directory, seqtk_directory=seqtk_directory, referenceGenome_file=referenceGenome_file, outdirectory=outdirectory, BED_filename_combined=intergene_read_bedfile_prename, synthetic_fastq_prename=intergene_read_bedfile_prename)
```

### Combine FASTQ files for genes and inter-genes

```{code-block} python3
synthetic_fastq_prename = "%s.syntheticBAM.combined" % filename
scRNA_GenerateBAM.combineFASTQ(gene_synthetic_fastq_prename=gene_read_bedfile_prename, intergene_synthetic_fastq_prename=intergene_read_bedfile_prename, outdirectory=outdirectory, synthetic_fastq_prename=synthetic_fastq_prename)
```