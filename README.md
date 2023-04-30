Contents
- [Overview](#overview)
  - [Quick Start](#quick-start)
  - [Background](#background)
  - [Pipeline](#pipeline)
- [Input Files](#input-files)
  1. [`config.yaml`](#config-yaml)
  2. [`samples.json`](#samples-json)
  3. [`assets/bpm.fasta`](#bpm-fasta)
  4. [`assets/dpm96.fasta`](#dpm-fasta)
  5. [`config.txt`](#config-txt)
  6. [`format.txt`](#format-txt)
  7. [`assets/blacklist_hg38.bed`, `assets/blacklist_mm10.bed`](#blacklist-bed)
  8. [`assets/index_mm10/*.bt2`, `assets/index_hg38/*.bt2`](#index-bt2)
- [Output Files](#output-files)

# Overview

## Quick Start

This pipeline assumes an existing [conda](https://conda.io) installation and is written as a [Snakemake](https://snakemake.github.io/) workflow. To install Snakemake with conda, run

```
conda env create -f envs/snakemake.yml
conda activate snakemake
```

to create and activate a conda environment named `snakemake`. Once all the [input files](#input-files) are ready, run the pipeline on a SLURM server environment with

```
./run_pipeline.sh
```

<!-- TODO: specify Java and Bash versions -->
<!-- TODO: pipeline for local laptop computer -->

## Background

Terms
- **barcode**: this term is overloaded to refer to one of two possible sequences
  - **split-pool barcode**: the permutation of split-pool tags that uniquely identifes a cluster
  - **antibody oligo barcode**: a 9 nt sequence within the antibody oligo that uniquely identifies a type of antibody

<!-- TODO: figures of expected sequences -->

## Pipeline

The pipeline relies on scripts written in Java, Bash, Python. This pipeline has been validated using Java version 8.0.322 (`openjdk version "1.8.0_322"`) and Bash version 4.2.46. Versions of Python are specified in conda environments described in `envs/`, along with other third-party programs and packages that this pipeline depends on.

1. Split FASTQ files into chunks for parallel processing
2. Adaptor trimming (Trim Galore!)
3. Barcode identification 
4. Split DPM (DNA) and BPM (antibody oligo) reads into separate files
5. DPM read workflow:
   1. DPM Trimming (cutadapt)
   2. Alignment (bowtie2)
   3. Chromosome relabeling (add "chr") and filtering (removing non-canonical chromosomes)
   4. Masking (based on ENCODE blacklists)
6. BPM read workflow:
   1. BPM Trimming (cutadapt)
   2. FASTQ to BAM conversion
7. Cluster generation
8. Cluster assignment and antibody specific BAM file creation
9. Summary plots
   1. DPM and BPM cluster size distributions
   2. Maximum representation oligo ECDFs
10. Summary statistics
	1. MultiQC (trimming, alignments)
	2. Ligation efficiency
	3. Cluster statistics
	4. Read assignment statistics

# Input Files

All paths are relative to the project directory.

1. <a name="config-yaml">`config.yaml`</a>: YAML file containing the processing settings and paths of required input files.

2. <a name="samples-json">`samples.json`</a>: JSON file with the location of FASTQ files (read1, read2) to process.
   - `config.yaml` key to specify the path to this file: `samples`
   - This can be prepared using `fastq2json.py --fastq_dir <path_to_directory_of_FASTQs>` or manually formatted as follows:
     
     ```{json}
     {
        "sample1": {
          "R1": ["<path_to_data>/sample1_S1_R1_001.fastq.gz"],
          "R2": ["<path_to_data>/sample1_S1_R2_001.fastq.gz"]
        },
        "sample2": {
          "R1": ["<path_to_data>/sample2_S2_R1_001.fastq.gz"],
          "R2": ["<path_to_data>/sample2_S2_R2_001.fastq.gz"]
        },
        ...
     }
     ```
     <br>
   - The pipeline (in particular, the script `scripts/bash/split_fastq.sh`) currently only supports one read 1 (R1) and one read 2 (R2) FASTQ file per sample.
     - If there are multiple FASTQ files per read orientation per sample (for example, if the same sample was sequenced multiple times, or it was split across multiple lanes during sequencing), the FASTQ files will first need to be concatenated together, and the paths to the concatenated FASTQ files should be supplied in the JSON file.

3. <a name="bpm-fasta">`assets/bpm.fasta`</a>: FASTA file containing the sequences of antibody oligo barcodes
   - `config.yaml` key to specify the path to this file: `cutadapt_oligos`
   - Used by: `cutadapt` (Snakefile `rule cutadapt_oligo`)
   - Each sequence should be preceeded by `^` to anchor the sequence during cutadapt trimming (see Snakefile `rule cutadapt_oligo`).

4. <a name="dpm-fasta">`assets/dpm96.fasta`</a>: FASTA file containing the sequences of DPM tags
   - `config.yaml` key to specify the path to this file: `cutadapt_dpm`
   - Used by: `cutadapt` (Snakefile `rule cutadapt_dpm`)
   - Each of these sequences are 10 nt long, consisting of a unique 9 nt DPM_Bottom sequences as originally designed for SPRITE (technically, only the first 8 nt are unique, and the 9th sequence is always a `T`), plus a `T` that is complementary to a 3' `A` added to a chromatin DNA sequence via dA-tailing.
<!--TODO: for chromatin read 1 - we are trimming the 5' DPM, but are we trimming the 3' DPM if the read extends beyond the DNA insert sequence? -->

5. <a name="config-txt">`config.txt`</a>: Text file containing the sequences of split-pool tags and the split-pool barcoding setup.
   - `config.yaml` key to specify the path to this file: `bID` (for "barcode ID")
   - Used by: `scripts/java/BarcodeIdentification_v1.2.0.jar` (Snakefile `rule barcode_id`) and `scripts/python/fastq_to_bam.py` (Snakefile `rule fastq_to_bam`)
   - Format: SPRITE configuration file (see our SPRITE [GitHub Wiki](https://github.com/GuttmanLab/sprite-pipeline/wiki/1.-Barcode-Identification#configuration-file) or [*Nature Protocols* paper](https://doi.org/10.1038/s41596-021-00633-y) for details).
     - Blank lines and lines starting with `#` are ignored.
     - An example barcoding configuration file is annotated below:

       ```
       # Barcoding layout for read 1 and read 2
       # - Y represents a terminal tag
       # - ODD, EVEN, and DPM indicate their respective tags
       # - SPACER accounts for the 7-nt sticky ends that allow ligation between tags
       READ1 = DPM
       READ2 = Y|SPACER|ODD|SPACER|EVEN|SPACER|ODD|SPACER|EVEN|SPACER|ODD
       
       # DPM tag sequences formatted as tab-delimited lines
       # 1. Tag category: DPM
       # 2. Tag name: must contain "DPM", such as "DPM<xxx>"
       # 3. Tag sequence (see assets/dpm96.fasta)
       # 4. Tag error tolerance: acceptable Hamming distance between
       #    expected tag sequence (column 3) and tag sequence in the read
       DPM	DPM6B1	TCGAGTCT	0
       DPM	DPM6B10	TGATGCAT	0
       ...
       
       # Antibody oligo barcode sequences formatted as tab-delimited lines
       # - Identical format as for DPM tag sequences, except that the tag name (column 2)
       #   must contain "BEAD", such as "BEAD_<name of antibody>"
       DPM	BEAD_AB1	GGAACAGTT	0
       DPM	BEAD_AB2	CGCCGAATT	0
       ...
       
       # Split-pool tag sequences: same 4-column tab-delimited format as the 
       #   "DPM and antibody oligo barcode sequences" section above, except that 
       #   Tag category (column 1) is now ODD, EVEN, or Y
       EVEN	A1	ATACTGCGGCTGACG	2
       EVEN	A2	GTAGGTTCTGGAATC	2
       ...
       ODD	A1	TTCGTGGAATCTAGC	2
       ODD	A2	CCTGTGCGTTAGAGT	2
       ...
       Y	A1	TATTATGGT	0
       Y	A2	GAGATGGAT	0
       ...
       ```
       <!-- TODO: why are the DPM sequences in the config.txt file trimmed compared to dpm96.fasta? -->

6. <a name="format-txt">`format.txt`</a>: Tab-delimited text file indicating which split-pool barcode tags are valid in which round of split-pool barcoding (i.e., at which positions in the barcoding string).
   - `config.yaml` key to specify the path to this file: `format`
   - Used by: `scripts/python/split_dpm_bpm_fq.py` (Snakefile `rule split_bpm_dpm`)
   - Column 1 indicates the zero-indexed position of the barcode string where a tag can be found.
     - Term barcode tags (Y) are position `0`; the second to last round of barcoding tags are position `1`; etc. A value of `-1` in the position column indicates that the barcode tag was not used in the experiment.
   - Column 2 indicates the name of the tag.
   - Column 3 is the tag sequence.
   - Column 4 is the edit acceptable edit distance for this sequence.
<!--TODO: verify exactly how format.txt is parsed-->

7. <a name="blacklist-bed">`assets/blacklist_hg38.bed`, `assets/blacklist_mm10.bed`</a>: blacklisted genomic regions for ChIP-seq data
   - For human genome release hg38, we use [ENCFF356LFX](https://www.encodeproject.org/files/ENCFF356LFX/) from ENCODE. For mouse genome release mm10, we use [mm10-blacklist.v2.bed.gz](https://github.com/Boyle-Lab/Blacklist/blob/master/lists/mm10-blacklist.v2.bed.gz).
   - Reference paper: Amemiya HM, Kundaje A, Boyle AP. The ENCODE Blacklist: Identification of Problematic Regions of the Genome. Sci Rep. 2019;9(1):9354. doi:10.1038/s41598-019-45839-z
   - Example code used to download them into the `assets/` directory:
   
     ```{bash}
     wget -O - https://www.encodeproject.org/files/ENCFF356LFX/@@download/ENCFF356LFX.bed.gz |
         zcat |
         sort -V -k1,3 > "assets/blacklist_hg38.bed"

     wget -O - https://github.com/Boyle-Lab/Blacklist/raw/master/lists/mm10-blacklist.v2.bed.gz |
         zcat |
         sort -V -k1,3 > "assets/blacklist_mm10.bed"
     ```

8. <a name="index-bt2">`assets/index_mm10/*.bt2`, `assets/index_hg38/*.bt2`</a>: Bowtie 2 genome index
   - `config.yaml` key to specify the path to the index: `bowtie2_index: {'mm10': <mm10_index_prefix>, 'hg38': <hg38_index_prefix>}`
   - If you do not have an existing Bowtie 2 index, you can download [pre-built indices](https://bowtie-bio.sourceforge.net/bowtie2/manual.shtml) from the Bowtie 2 developers:

     ```{bash}
     # for human primary assembly hg38
     mkdir -p assets/index_hg38
     wget https://genome-idx.s3.amazonaws.com/bt/GRCh38_noalt_as.zip
     unzip -d assets/index_hg38 GRCh38_noalt_as.zip \*.bt2

     # for mouse primary assembly mm10
     mkdir -p assets/index_mm10
     wget https://genome-idx.s3.amazonaws.com/bt/mm10.zip
     unzip -d assets/index_mm10 mm10.zip
     ```
     <br>
     This will create a set of files under `assets/index_hg38` or `assets/index_mm10`. If we want to use the `mm10` genome assembly, for example, the code above will populate `assets/index_mm10` with the following files: `mm10.1.bt2`, `mm10.2.bt2`, `mm10.3.bt2`, `mm10.4.bt2`, `mm10.rev.1.bt2`, `mm10.rev.2.bt2`. The path prefix to this index (as accepted by the `bowtie2 -x <bt2-idx>` argument) is therefore `assets/index_mm10/mm10`, which is set in the configuration file, `config.yaml`.

# Output Files

1. Barcode Identification Efficiency (`workup/ligation_efficiency.txt`)
   - A statistical summary of how many barcode tags were found per read and the proportion of reads with a matching barcode at each barcode position.

2. Clusterfile (`workup/clusters/<sample>.clusters`)
   - Each line in a cluster file represents a single cluster. The first column is the cluster barcode. The remainder of the line is a tab deliminated list of reads. DNA reads are formated as `DPM[strand]_chr:start-end` and Antibody ID oligo reads are formated as `BPM[]_<AntibodyID>:<UMI>-0`.

3. Cluster statistics (`workup/clusters/cluster_statistics.txt`)
   - The number of clusters and BPM or DPM reads per library.

4. Cluster size distribtion (`workup/clusters/[BPM,DPM]_cluster_distribution.pdf`)
   - The distribution showing the proportion of clusters that belong to each size category.

5. Cluster size read distribution (`workup/clusters/[BPM,DPM]_read_distribution.pdf`)
   - The distribution showing the proportion of reads that belong to clusters of each size category. This can be more useful than the number of clusters since relatively few large clusters can contain many sequencing reads (i.e., a large fraction of the library) while many small clusters will contain few sequencing reads (i.e., a much smaller fraction of the library).
   
6. Maximum Representation Oligo ECDF (`workup/clusters/Max_representation_ecdf.pdf`)
   - A plot showing the distribution of proportion of BPM reads in each cluster that belong to the maximum represented Antibody ID in that cluster. A successful experiment should have an ECDF close to a right angle. Deviations from this indicate that beads contain mixtures of antibody ID oligos. Understanding the uniqueness of Antibody ID reads per cluster is important for choosing the thresholding parameters (`min_oligo`, `proportion`) for cluster assignment.

7. Maximum Representation Oligo Counts ECDF (`workup/clusters/Max_representation_ecdf.pdf`)
   - A plot showing the distribution of number of BPM reads in each cluster that belong to the maximum represented Antibody ID in that cluster. If clusters are nearly unique in Antibody ID composition, this plot is a surrogate for BPM size distribtuion. Understanding the number of Antibody ID reads per cluster is important for choosing the thresholding parameters (`min_oligo`, `proportion`) for cluster assignment.

8. BAM Files for Individual Antibodies (`workup/splitbams/*.bam`)
   - Thresholding criteria (`min_oligos`, `proportion`, `max_size`) for assigning individual clusters to individual antibodies are set in [`config.yaml`](#config-yaml). The "none" BAM file contains DNA reads from clusters without Antibody ID reads. THe "ambigious" BAM file contains DNA reads from clusters that failed the proportion thresholding criteria. The "uncertain" BAM file contains DNA reads from clusters that failed the min_oligo thresholding criteria. The "filtered" BAM file contains DNA reads from clusters that failed the max_size thresholding criteria.

9. Read Count Summary for Individual Antibodies (`workup/splitbams/splitbam_statistics.txt`)
   - The number of read counts contained within each individual BAM file assigned to individual antibodies.

# Credits

Adapted from the [SPRITE](https://github.com/GuttmanLab/sprite-pipeline) and [RNA-DNA SPRITE](https://github.com/GuttmanLab/sprite2.0-pipeline) pipelines by **Isabel Goronzy** ([@igoronzy](https://github.com/igoronzy)).

Other contributors
- Benjamin Yeh ([@bentyeh](https://github.com/))
- Andrew Perez
- Mario Blanco ([@mrblanco](https://github.com/mrblanco))
- Mitchell Guttman ([@mitchguttman](https://github.com/mitchguttman))
