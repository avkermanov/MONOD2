# MONOD2
MONOD2 is a toolkit for methylation haplotype analysis for bisulfite sequencing or (BS-seq) data using high-throughput sequencing.  The analysis can be divided into three parts. First sequence alignment files are analyzed to generate methylation haplotypes which includes haplotype strings and haplotype counts for CpG positions with coverage.  Next, a collection of methylation haplotype files can be analyzed for identification of methylation haplotype blocks or for differential methylation haplotype load. 

MONOD2 is comprised of executable `sh`, `perl`, and `R` programs. No installation is necessary except for the required dependencies listed below.

## Contact
email: hdinhdp@gmail.com

## Download

You can use git to download the entire toolkit. 

```
git clone https://github.com/dinhdiep/MONOD2

```

## Required R packages

You can install all the R packages from command line in R with the following:

```
require(gplots)
require(MASS)
require(pheatmap)
require(RColorBrewer)
require(randomForest)
require(impute)
require(abind)
require(ggplot2)
require(earth)
require(skmeans)
require(ModelMetrics)
require(caret)
require(ROCR)
require(gplot2)
require(reshape)
require(compiler)

```

## Required pre-installed software

1. [samtools version 1.2 or above](http://samtools.sourceforge.net/)
2. [bedtools version 2.26 or above](http://bedtools.readthedocs.io/en/latest/)

## Important Notes

MONOD2 is based on the idea described in Guo et al. 2017 Nature Genetics (doi:10.1038/ng.3805). Here, we have made significant modifications from the original methods and codes that was hosted on the [Supplementary Website](http://genome-tech.ucsd.edu/public/MONOD_NG_TR44413). Some notable modifications are listed below:

1. The original code for Figure 2 was not producing the correct values from the Figure. Note that there are two errors in Figure 2 as follows: the epipolymorphism for panel 4 should be 0.9375 (the published figure have 0.375 due to mis-editing), the MHL value for panel 5 should be 0.1167 (the published figure have a rounded up value of 0.1200).  

2. The original comparison of AMF versus MHL at tissue specific regions were unfair because they used differentially methylated regions identified by MHL. In MONOD2 we use only the MHB regions overlapping with published tissue specific DMRs to compare AMF versus MHL.

3. The original method for tumor load estimation performed features selection without separation of test data (the plasma samples). In MONOD2 we identified the features using a subset of 50 normal plasma only, and test on the remaining normal and cancer patient plasma samples.

4. The original method for methylation haplotype load analysis of cf-DNA did not have proper separation of training and test data for features selection although in model building the training and test data were separated. We realized that this would heavily bias the results to the sample set. In MONOD2 we separated training and test data for both features selection and model building.

## Usage

### Extracting methylation haplotypes from sequence alignment files

Sequence alignment files in BAM format should have the expected sam flags for Watson (forward) and Crick (reverse) strands. The code below will generate CpG haplotype files without clonal removal (any clonal remove must be done upstream).
```
bam2cghap.sh [cpg position file] [bam file] [output file prefix name]

```

Run example
```
./scripts/bam2cghap.sh allcpg/cpg.small.txt.gz BAMfiles/Colon_primary_tumor_sept9_promoter.bam test

```
The version 1 code is provided but user must specify RRBS or WGBS mode and can only use BAM that have been mapped to hg19. For RRBS and WGBS data are treated differently with the major difference in that reads which are considered clonal in WGBS would be removed in order to report only one haplotype.

Run RRBS example

```
./scripts/bam2cghap_v1.sh RRBS Colon_primary_tumor_sept9_promoter.RD1_80up.genomecov.bed allcpg/cpg.small.txt.gz BAMfiles/Colon_primary_tumor_sept9_promoter.bam test

```
Run WGBS example

```
./scripts/bam2cghap_v1.sh WGBS Colon_primary_tumor_sept9_promoter.RD1_80up.genomecov.bed allcpg/cpg.small.txt.gz BAMfiles/Colon_primary_tumor_sept9_promoter.bam test

```

### Making mappable bin file

Empirically determine which regions in the genome have high mappability using whole genome bisulfite sequencing datasets. 

```
make-mappable-bins.sh [bam file] [minimum depth cutoff]

```

Run example

```
./scripts/make-mappable-bins.sh BAMfiles/Colon_primary_tumor_sept9_promoter.bam 5

```

### Identifying the methylation haplotype blocks 

Methylation haplotypes were split into continuous mappable bins and then an algorithm greedily select the largest possible continuous region with a minimum linkage disequilibrium score to be considered methylation haplotype blocks. Blocks must have at least 3 CpGs sites. For more info about methylation linkage disequilibrium, please see [Shoemaker et al. 2010 Genome Research](http://genome.cshlp.org/content/20/7/883.full.pdf).

```
cghap2mhbs.sh [haplotype file] [target bed] [minimum LD R2 cutoff] [output name prefix]

```
Run example

```
./scripts/cghap2mhbs.sh HaploInfo/chr22.sub.hapInfo.txt example/N37_10_tissue_pooled.autosomes.RD10_80up.genomecov.bed 0.3 chr22

```

### Compare AMF versus MHL 

We can compare the signal to noise levels between a matrix calculated using the average methylation frequency (AMF) and a matrix calculated using the methylation haplotype load (MHL). The three required files are as followed.
1. WGBS MHL matrix. Example: ng.3805/WGBS.getHaplo.mhl.mhbs1.0.rmdup_consistent.useSampleID.txt.gz
2. WGBS AMF matrix. Example: ng.3805/WGBS.getHaplo.amf.mhbs1.0.rmdup_consistent.useSampleID.txt.gz
3. The published DMRs file. Example: ng.3805/RRBS_MHBs.sorted.DMR.withID.bed

The only requirements for (1) and (2) are that the region indices and the sample IDs are the same between the two matrices. The region file (3) is a list of regions (subset of the region indices used to make the matrices). 

Run example

```
./scripts/mhl_vs_amf.sh

```

The output is an Rplots.pdf file that includes two heatmaps similar to Figure 3 from Guo et al. 2017, while subsetting at regions with tissue specific DMRs and also includes a scatteplot showing the relative signal to noise levels for AMF versus MHL.


### Preprocess the data matrices 

Analysis cannot be performed across RRBS and WGBS datasets due to techical artifacts that may bias the results. Therefore we generate two separate matrices, one for each dataset. We then perform data pruning where samples with too few region coverage and regions with too few sample coverage are removed. After pruning, imputation using k-nearest neighbor to fill all the missing values is performed. To run the provided preprocessing script, the following files are required.

1. The metadata for the WGBS tissue samples: ng.3805/WGBS.getHaplo.sampleInfo.txt
2. The metadata for the RRBS tissue/plasma samples: ng.3805/RRBS.getHaplo.sampleInfo.txt
3. WGBS MHL matrix: ng.3805/WGBS.getHaplo.mhl.mhbs1.0.rmdup_consistent.useSampleID.txt
4. RRBS MHL matrix: ng.3805/RRBS_170609.gethaplo.mhl.mhbs1.0.useSampleID.txt

Run example

```
./scripts/preprocess.sh ng.3805/RRBS_170609.gethaplo.mhl.mhbs1.0.useSampleID.txt.gz ng.3805/WGBS.getHaplo.mhl.mhbs1.0.rmdup_consistent.useSampleID.txt.gz ng.3805/RRBS.getHaplo.sampleInfo.txt ng.3805/WGBS.getHaplo.sampleInfo.txt wgbs_rrbs_clean.Rdata

```

The output is an R data file named 'wgbs_rrbs_clean.Rdata' and several PDF files showing the data quality.

### Tumor load estimation

First, we generate the simulation files by merging a subset of normal plasma data with cancer tissue data. We have both primary tumor tissue data for lung cancer and colon cancer. Since the large files cannot be hosted on github, we randomly sampled (with replacement) the datasets and provided them in the BAMfiles folder. Note that read IDs can contain either mapped reads or all reads. We performed the analysis on all reads so that the mappability of sampled fragments also reflect the mappability of 'non-cancer' versus 'cancer' fragments. 

The script needs to be modified from within (using a text editor) in order to run user-provided read IDs and BAM files. The following lines as is will perform the analysis on the example read IDs and BAM files. 

```
numsimulation=20 
n1=1000000 

cctfastq="BAMfiles/CCT.readIDs.txt"
lctfastq="BAMfiles/LCT.readIDs.txt"
ncpfastq="BAMfiles/NCP.readIDs.txt"

cctbam="BAMfiles/CCT.bam"
lctbam="BAMfiles/LCT.bam"
ncpbam="BAMfiles/NCP.bam"

cpgs="allcpg/hg19.fa.allcpgs.txt.gz"

```
Run example

```
 ./scripts/simulate-cghap.sh

```

There should be many simulated files generated including two list files: (1) cct.hapinfo.list_20 (2) lct.hapinfo.list_20 which are required as input to the next step. The should also be three directories (1) CCT_simulation (2) LCT_simulation and (3) NCP_simulation which contain simulated data for each sample type.

Finally, we can run features selection and tumor load estimation using the simulated data files. The script for tumor load estimation have the following usage.

```
tumor_load_estimation.sh [rdata file] [colon cancer simulation table] [lung cancer simulation table] [list of normal plasma training samples] [number of simulations]

```

Run example

```
./scripts/tumor_load_estimation.sh wgbs_rrbs_clean.Rdata cct.hapinfo.list_20 lct.hapinfo.list_20 ng.3805/subset_normal_plasma_training 20

```

The output files are as follows.

1. The Rdata which saves all the outputs : Tumor_load_estimation.Rdata
2. Boxplots similar to Figure 4d from Guo et al. 2017 which shows the estimated tumor proportions for the plasma samples: estimated_proportions.pdf
3. Boxplots similar to Figure 4a,b from Guo et al. 2017 which shows the differential MHL levels in tumor marker regions for different sample types: mhl_boxplot.pdf
4. Heatmaps similar to Figure 4a,b from Guo et al. 2017 which shows the MHL levels in normal plasma, cancer plasma, and cancer tissues in the tumor marker regions: heatmap.pdf
5. A table of the standard curve values generated from simulated data: standard_curves_values.txt
6. A list of lung cancer markers used in the analysis: lung_cancer_markers.txt
7. A list of colon cancer markers used in the analysis: colon_cancer_markers.txt


### Plasma prediction

Run example

```
./scripts/analyze-cf-dna.sh

```

