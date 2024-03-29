# Long read assembly #

## Table of Contents ##

1. [Performs high accuracy basecalling from FAST5 files](#basecalling) ([**guppy**](https://community.nanoporetech.com/protocols/Guppy-protocol/v/gpb_2003_v1_revaa_14dec2018/linux-guppy))
2. [Trim the adapters, barcodes and poor-quality bases](#trim) ([**guppy**](https://community.nanoporetech.com/protocols/Guppy-protocol/v/gpb_2003_v1_revaa_14dec2018/linux-guppy))
3. [(optional) base quality correction](#correction) ([**LorDEC**](http://www.atgc-montpellier.fr/lordec/))
4. [Assemble reads](#flye) ([**Flye**](https://github.com/fenderglass/Flye))
5. [Polish and imporve assembly](#racon) ([**Racon**](https://github.com/isovic/racon), [**minimap2**](https://github.com/lh3/minimap2))
6. [Genomic assemblies evaluation and comparison](#quast) [(**QUAST**)](https://github.com/ablab/quast)
7. [Calculating average coverage of the genome](#average_coverage) ([**BWA**](https://github.com/lh3/bwa) and [**Samtools**](https://github.com/samtools/samtools))
8. [automation of the process](#automation)

<a name = "basecalling"></a>
### Performs high accuracy basecalling from FAST5 files (guppy) ###
<a name = "trim"></a>
### Trim the adapters, barcodes and poor-quality bases (guppy) ###

Basecalling is the process of generating sequence data with its base quality score (.fastq) from the raw signaling results of the Nanopore MinION sequencer. Guppy is the current basecaller provided from Oxford Nanopore. Basecalling can be completed real-time during the sequencing run, however we can also generate high accuracy reads based on the complicated neural network based basecalling from Guppy. Additionally, Guppy can be used to trim the low quality reads and barcode sequences during basecalling

Input files format - *.fast5*
output files format - *.fastq*

before starts, please check if guppy directory is in your PATH varaibles
```
export PATH=$PATH:/directory for guppy installation/bin
echo $PATH #check if ~/ont-guppy-cpu/bin is in the PATH variable
```

Oxford Nanaopore technology provide different type of sequencing library kits, and different flowcells from different technology. It is important to select the same kits/flowcells for basecalling to recognize the right signal. To check the list of the kits and flowcells
```
guppy_basecaller --print_workflows
```
Now you will see something like this
```
Available flowcell + kit combinations are:
flowcell       kit               barcoding config_name                    model version
FLO-PRO001     SQK-LSK109                  dna_r9.4.1_450bps_hac_prom     2021-05-05_dna_r9.4.1_promethion_384_dd219f32
FLO-PRO001     SQK-LSK109-XL               dna_r9.4.1_450bps_hac_prom     2021-05-05_dna_r9.4.1_promethion_384_dd219f32
FLO-PRO001     SQK-LSK110                  dna_r9.4.1_450bps_hac_prom     2021-05-05_dna_r9.4.1_promethion_384_dd219f32
FLO-PRO001     SQK-DCS109                  dna_r9.4.1_450bps_hac_prom     2021-05-05_dna_r9.4.1_promethion_384_dd219f32
FLO-PRO001     SQK-PCS109                  dna_r9.4.1_450bps_hac_prom     2021-05-05_dna_r9.4.1_promethion_384_dd219f32
FLO-PRO001     SQK-PCS110                  dna_r9.4.1_450bps_hac_prom     2021-05-05_dna_r9.4.1_promethion_384_dd219f32
FLO-PRO001     SQK-PRC109                  dna_r9.4.1_450bps_hac_prom     2021-05-05_dna_r9.4.1_promethion_384_dd219f32
FLO-PRO001     SQK-MLK110-96-XL  included  dna_r9.4.1_450bps_hac_prom     2021-05-05_dna_r9.4.1_promethion_384_dd219f32
FLO-PRO001     SQK-PCB109        included  dna_r9.4.1_450bps_hac_prom     2021-05-05_dna_r9.4.1_promethion_384_dd219f32
FLO-PRO001     SQK-PCB110        included  dna_r9.4.1_450bps_hac_prom     2021-05-05_dna_r9.4.1_promethion_384_dd219f32
FLO-PRO002     SQK-LSK109                  dna_r9.4.1_450bps_hac_prom     2021-05-05_dna_r9.4.1_promethion_384_dd219f32
FLO-PRO002     SQK-LSK109-XL               dna_r9.4.1_450bps_hac_prom     2021-05-05_dna_r9.4.1_promethion_384_dd219f32
FLO-PRO002     SQK-LSK110                  dna_r9.4.1_450bps_hac_prom     2021-05-05_dna_r9.4.1_promethion_384_dd219f32
```

Find the right combination between flowcells and library preparation kits (here, we are using "SQK-RBK004" and "FLO-MIN106"

```
flowcell       kit               barcoding config_name                    model version
FLO-MIN106     SQK-RBK004        included  dna_r9.4.1_450bps_hac          2021-05-17_dna_r9.4.1_minion_384_d37a2ab9
```

Now, specify the input file, output directory, and configuration name (don't forget to add configuration file extenstion (.cfg) at the end)

**hac** is representing "high accuracy"

```
guppy_basecaller -i [input.fast5] -s [output directory] 
                 -c dna_r9.4.1_450bps_hac.cfg --num_callers 2 --cpu_threads_per_caller 1 --trim_barcodes
```
```
--num_callers : how many parallel basecallers to create
--cpu_threads_per_caller : how many threads will be used per each callers

[num_callers] * [cpu_threads_per_caller] = number of available threads

--trim_barcodes : barcode trimming based on the kit information provided
```

Now, it will generate multiple *.fastq* file from *.fast5* file in our output directory, you can simply combined all the sequences from *.fastq* files into one *.fastq* file
```
cd [output directory]
cat *.fastq > [output file name].fastq
```

Now, one huge *.fastq* file is generated as [output file name].fastq

<a name = "correction"></a>
### (optional) base quality correction (LorDEC) ###
* This step only can be completed if you also have illumina short read sequences as a reference

LorDEC is a program for error correcting in long reads.Since it uses a hybrid strategy, you need the reference read set - whose error rate is low, and the long-read sequence data, which is corrected by the reference set.

<a name = "flye"></a>
### Assemble reads (Flye) ###

Flye is a *de novo* assembler for long read sequencing reads (i.e., PacBio or Oxford Nanopore). It takes raw reads as input and outputs polished contigs. 

If you are using raw nanopore reads directly from the run (fast base balled reads)
```
flye --nano-raw [combined raw reads from previous step].fastq --genome-size 5m -o ./ -t 4
```
If you are using corrected high accuracy reads from Guppy output
```
flye --nano-hq [combined raw reads from previous step].fastq --genome-size 5m -o ./ -t 4
```

<a name = "racon"></a>
### Polish and imporve assembly (Racon, minimap2 and miniasm) ###

In this step, we will use minimap2 to align sequence read against the contigs first, and it will be used for assembly polishing by Racon. 

#### Run minimap2 for alignment ####
```
minimap2 -x map-ont -t 4 [output from flye assembler].fasta [Nanopore raw reads].fastq > temporary_path
```

#### Run Racon for polishing ####
```
racon -t 4 [Nanopore raw reads].fastq temporary_path [output from flye assembler].fasta > polished_contigs.fasta
```

<a name = "quast"></a>
### Genomic assemblies evaluation and comparison (QUAST) ###

QUAST (Quality Assessment Tool for Genome Assemblies) is a widely used bioinformatics tool for evaluating genome assemblies. A number of important output will be used to evaluate overall assemlies quality (e.g., N50, Number of contigs, Total length of the assembly, GC%).

![image](https://user-images.githubusercontent.com/62360632/143988245-29693950-a04d-4510-9501-ec9120871451.png)

**N50** is the shortest contig length that needs to be included for covering 50% of the genome. Meaning, Half of the genome sequence is covered by contigs larger than or equal the N50 contig size. Meaning, The sum of the lengths of all contigs of size N50 or longer contain at least 50 percent of the total genome sequence.

**GC-content** (or guanine-cytosine content) is the percentage of nitrogenous bases in a DNA or RNA molecule that are either guanine (G) or cytosine (C). This measure indicates the proportion of G and C bases out of an implied four total bases, also including adenine and thymine in DNA and adenine and uracil in RNA.

#### Running QUAST ####
```
quast -o [output directory] -i polished_contigs.fasta 
```
#### Output files ####
```
report.txt      summary table
report.tsv      tab-separated version, for parsing, or for spreadsheets (Google Docs, Excel, etc)  
report.tex      Latex version
report.pdf      PDF version, includes all tables and plots for some statistics
report.html     everything in an interactive HTML file
```

<a name = "average_coverage"></a>
### Calculating average coverage of the genome (BWA and Samtools) ###

Calculating average coverage of the genome is a process to align raw reads to the assembled contig. By doing this, we can calculate the average coverage by counting number of cumulate based in each position on the contigs.

First off, assembled contig need to be 'indexed', meaning that those sequences will be used as reference in calculating average coverage. Next, raw trimmed reads will be mapped to the indexed reference by bwa. Lastly, calculate average coverage after sorting those mapped reads. 

```
# Indexing contigs
bwa index <contigs.fasta>

# Mapping reads to the indexed contigs
bwa mem -t <number of threads> <contig.fasta file> <trimmed reads file 1 .fastq> <trimmed reads file .fastq> > <output>.sam

# Converting SAM file to BAM with samtools
samtools view -Sb <output>.sam -o <output>.bam
rm *.sam

#Sorting BAM file with samtools
samtools sort <output>.bam <output>_sorted.bam

#Indexing sorted BAM file
samtools index <output>_sorted.bam

#Finally to calculate average coverage
X=$(samtools depth $<output>_sorted.bam | awk '{sum+=$3} END { print sum/NR}');
echo "<output>_sorted.bam";
echo "$X";
echo "<output>_sorted.bam $X" >> average_coverage.txt
```


<a name = "automation"></a>
### Automation of the process
