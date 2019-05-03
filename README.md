# RNA-Sequence-Analysis-Non-Model-Species-Eastern-larch

# RNA Sequence Analysis for Non Model Species Eastern larch (Tamarack)  
  
This repository is a usable, publicly available tutorial for analyzing differential expression data and creating topological gene networks. All steps have been provided for the UConn CBC Xanadu cluster here with appropriate headers for the Slurm scheduler that can be modified simply to run.  Commands should never be executed on the submit nodes of any HPC machine.  If working on the Xanadu cluster, you should use sbatch scriptname after modifying the script for each stage.  Basic editing of all scripts can be performed on the server with tools such as nano, vim, or emacs.  If you are new to Linux, please use [this](https://bioinformatics.uconn.edu/unix-basics) handy guide for the operating system commands.  In this guide, you will be working with common bio Informatic file formats, such as [FASTA](https://en.wikipedia.org/wiki/FASTA_format), [FASTQ](https://en.wikipedia.org/wiki/FASTQ_format), [SAM/BAM](https://en.wikipedia.org/wiki/SAM_(file_format)), and [GFF3/GTF](https://en.wikipedia.org/wiki/General_feature_format). You can learn even more about each file format [here](https://bioinformatics.uconn.edu/resources-and-events/tutorials/file-formats-tutorial/). If you do not have a Xanadu account and are an affiliate of UConn/UCHC, please apply for one **[here](https://bioinformatics.uconn.edu/contact-us/)**.  
  
Contents  
1. [Introduction](#1-introduction)  
2. [Quality Control](#2-quality-control)   
3. [Assembling the Transcriptome](#3-assembling-the-transcriptome)  
4. [Determining and removing repeat modules](#4-determining-and-removing-repeat-modules)   
 
 

## 1. Introduction  
  
In this tutorial we will be analyzing RNA-Sequence data from abscission zone tissue (between the needle and the stem) samples from the Eastern larch. The study is designed to examining the process of needle loss in Autumn. This data is not published and therefore can only be accessed through the Xanadu directory in "/UCHC/PublicShare/RNASeq_Workshop/Eastern_larch" We will be using the Eastern larch as a "non-model" organism.  
  
When an organism is called "model" there is an underlying assumption that very generous amounts of research have been performed on the species resulting in large pools of publicly available data. In biology and bioinformatics this means there are reference transcriptomes, structural annotations, known variant genotypes, and a wealth of other useful information in computational research. By contrast, when an organism is called "non-model" there is the underlying assumption that the information described prior will have to be generated by the research. This means that after extracting genetic data from a non-model organism, the researcher will then have to assemble the transcriptome, annotate the transcriptome, identify any pertinent genetic relationships, and so on. We can use this to develop a small map of our goals for analyzing our Eastern larch RNA samples. That is:  

![ out line ](/images/outline_wide.png)  

The data consists of 4 libraries under three different time points (roughly one month apart):  
*  U32 : UConn Tree 3, at time point 2  
*  U13 : UConn Tree 1, at time point 3  
*  K32 : Killingworth Tree 2, at time point 2   
*  K23 : Killingworth Tree 2, at time point 3

  
  
In this workflow we have seperated each step into folders, where you can find the appropriate scripts in conjunction with each steps. When you clone the git repository, the below directory structure will be cloned into your working directory.   

```  
Eastern_larch/
├── Raw_Reads
├── Quality_Control
├── Assembly
```  
   
   
The tutorial will be using SLURM schedular to submit jobs to Xanadu cluster. In each script we will be using it will contain a header section which will allocate the resources for the SLURM schedular. The header section will contain:  

```bash
#!/bin/bash
#SBATCH --job-name=JOBNAME
#SBATCH -n 1
#SBATCH -N 1
#SBATCH -c 1
#SBATCH --mem=1G
#SBATCH --partition=general
#SBATCH --qos=general
#SBATCH --mail-type=ALL
#SBATCH --mail-user=first.last@uconn.edu
#SBATCH -o %x_%j.out
#SBATCH -e %x_%j.err
```  

Before beginning, we need to understand a few aspects of the Xanadu server. When first logging into Xanadu from your local terminal, you will be connected to the submit node. The submit node is the interface with which users on Xanadu may submit their processes to the desired compute nodes, which will run the process. Never, under any circumstance, run processes directly in the submit node. Your process will be killed and all of your work lost! This tutorial will not teach you shell script configuration to submit your tasks on Xanadu. Therefore, before moving on, read and master the topics covered in the [Xanadu tutorial](https://bioinformatics.uconn.edu/resources-and-events/tutorials-2/xanadu/).  
  
  
Raw_Reads folder will look like:  
```
Raw_Reads/
├── K23
│   ├── K23_R1.fastq
│   └── K23_R2.fastq
├── K32
│   ├── K32_R1.fastq
│   └── K32_R2.fastq
├── U13
│   ├── U13_R1.fastq
│   └── U13_R2.fastq
└── U32
    ├── U32_R1.fastq
    └── U32_R2.fastq 
```
   
### Familiarizing yourself with the raw reads

The reads with which we will be working have been sequenced using [Illumina](https://www.illumina.com/techniques/sequencing.html). We assume that you are familiar with the sequencing technology. Let's have a look at the content of one of our reads, which are in the "fastq" format:

```bash
head -n 4 K32_R1.fastq
```

which will show the first four lines in the fastq file:

```
@NS500402:381:HH3NFBGX9:1:11101:2166:1038 1:N:0:CGCTCATT+AGGCTATA
AGAACTCGAAACTAAACGTGGACGTGNTNNTATAAACNNANACNAATCCATCGCCGGTTNNCNTATNNNNNNNNNN
+
AAAAAEEEEEEEEEEEEEEEEEEEEE#E##EEEEEEE##E#EE#EEEE6EEEEEEEEEE##A#EAE##########
```

In here we see that first line corrosponds to the sample information followed by the length of the read, and in the second line corrosponds to the nucleotide reads, followed by the "+" sign where if repeats the information in the first line. Then the fourth line corrosponds to the quality score for each nucleotide in the first line.   
   
   
## 2. Quality Control

### Quality control of Illumina reads using Sickle
Step one is to perform quality control on the reads, and we will be using Sickle for the Illumina reads. To start with we have paired-end reads.  

```bash
module load sickle/1.33

sickle pe -f ../Raw_Reads/U13/U13_R1.fastq \
        -r ../Raw_Reads/U13/U13_R2.fastq \
        -t sanger \
        -o trim_U13_R1.fastq \
        -p trim_U13_R2.fastq \
        -s singles_U13.fastq \
        -q 30 -l 45

sickle pe -f ../Raw_Reads/U32/U32_R1.fastq \
        -r ../Raw_Reads/U32/U32_R2.fastq \
        -t sanger \
        -o trim_U32_R1.fastq \
        -p trim_U32_R2.fastq \
        -s singles_U32.fastq \
        -q 30 -l 45

sickle pe -f ../Raw_Reads/K32/K32_R1.fastq \
        -r ../Raw_Reads/K32/K32_R2.fastq \
        -t sanger \
        -o trim_K32_R1.fastq \
        -p trim_K32_R2.fastq \
        -s singles_K32.fastq \
        -q 30 -l 45

sickle pe -f ../Raw_Reads/K23/K23_R1.fastq \
        -r ../Raw_Reads/K23/K23_R2.fastq \
        -t sanger \
        -o trim_K23_R1.fastq \
        -p trim_K23_R2.fastq \
        -s singles_K23.fastq \
        -q 30 -l 45
```
   
The useage information on the sickle program:  
```
Usage: sickle pe [options] -f <paired-end forward fastq file> 
	-r <paired-end reverse fastq file> 
	-t <quality type> 
	-o <trimmed PE forward file> 
	-p <trimmed PE reverse file> 
	-s <trimmed singles file>    

Options:
-f, --pe-file1, Input paired-end forward fastq file
-r, --pe-file2, Input paired-end reverse fastq file
-o, --output-pe1, Output trimmed forward fastq file
-p, --output-pe2, Output trimmed reverse fastq file
-s                Singles files

Global options:
-t, --qual-type, Type of quality values
                solexa (CASAVA < 1.3)
                illumina (CASAVA 1.3 to 1.7)
                sanger (which is CASAVA >= 1.8)
-s, --output-single, Output trimmed singles fastq file
-l, --length-threshold, Threshold to keep a read based on length after trimming. Default 20
-q, --qual-threshold, Threshold for trimming based on average quality in a window. Default 20
```  
   
The quality may be any score from 0 to 40. The default of 20 is much too low for a robust analysis. We want to select only reads with a quality of 30 or better. Additionally, the desired length of each read is 35bp. Again, we see that a default of 20 is much too low for analysis confidence. Lastly, we must know the scoring type. While the quality type is not listed on the SRA pages, most SRA reads use the "sanger" quality type. Unless explicitly stated, try running sickle using the sanger qualities. If an error is returned, try illumina. If another error is returned, lastly try solexa.  

The full slurm script which is called [sickle.sh](/Quality_Control/sickle.sh) is stored in the Quality_Control folder.  

At the end of the run, each run will produce **3** files, a trimmed forward read file, trimmed reverse read file and a singles file. Singles file will contain the reads which did not have a paired read to start with. The following files will be produced at the end of the run:  
```
Quality_Control/
├── trim_U13_R1.fastq
├── trim_U13_R2.fastq
├── singles_U13.fastq
├── trim_U32_R1.fastq
├── trim_U32_R2.fastq
├── singles_U32.fastq
├── trim_K23_R1.fastq
├── trim_K23_R2.fastq
├── singles_K23.fastq
├── trim_K32_R1.fastq
├── trim_K32_R2.fastq
└── singles_K32.fastq
```
  
The summary of the reads will be in the `*.out` file, which will give how many reads is kept and how many have been discarded in each run.  
  
| Sample | Input records | Paired records kept | single records kept | paired records discarded | single records discarded | Kept (%) |   
| --- | --- | --- | --- | --- | --- | --- |   
| U13 | 36516384 | 36516384 | 4048114 | 4004868 | 4048114 | 75.1 |   
| U32 | 46566276 | 35981128 | 3388161 | 3808826 | 3388161 | 77.3 |   
| K32 | 41656220 | 30657748 | 3646736 | 3705000 | 3646736 | 73.6 |     
| K23 | 45017196 | 33692758 | 3669578 | 3985282 | 3669578 | 74.2 |   
   
   
   
       
## 3. Assembling the Transcriptome   
    
### De novo Assembling the Transcriptome using Trinity   
   
Now that we've performed quality control we are ready to assemble our transcriptome using the RNA-Seq reads. We will be using the software [Trinity](https://github.com/trinityrnaseq/trinityrnaseq/wiki). Nearly all transcriptome assembly software operates under the same premise. Consider the following:

Suppose we have the following reads:
```
A C G A C G T T T G A G A
T T G A G A T T A C C T A G
```

We notice that the end of each read is the beginning of the next read, so we assemble them as one sequence by matching the overlaps:
```
A C G A C G T T T G A G A
              T T G A G A T T A C C T A G
```

Which gives us:
```
A C G A C G T [T T G A G A] T T A C C T A G
```

    
### De novo Assembling the Transcriptome using Trinity

In De novo assembly section, we will be woking in the `assembly` directory. In here we will be assembling the trimmed illumina reads seperatly using the trinity transcriptome assembler. Assembly requires a great deal of memory (RAM) and can take few days if the read set is large. Following is the trinity command that we use to assemble each transcriptome seperatly.    
   
```bash
module load trinity/2.6.6

Trinity --seqType fq \
        --left ../Quality_Control/trim_U13_R1.fastq \
        --right ../Quality_Control/trim_U13_R2.fastq \
        --min_contig_length 300 \
        --CPU 36 \
        --max_memory 100G \
        --output trinity_U13 \
        --full_cleanup 

Trinity --seqType fq \
        --left ../Quality_Control/trim_U32_R1.fastq \
        --right ../Quality_Control/trim_U32_R2.fastq \
        --min_contig_length 300 \
        --CPU 36 \
        --max_memory 100G \
        --output trinity_U32 \
        --full_cleanup

Trinity --seqType fq \
        --left ../Quality_Control/trim_K32_R1.fastq \
        --right ../Quality_Control/trim_K32_R2.fastq \
        --min_contig_length 300 \
        --CPU 36 \
        --max_memory 100G \
        --output trinity_K32 \
        --full_cleanup

Trinity --seqType fq \
        --left ../Quality_Control/trim_K23_R1.fastq \
        --right ../Quality_Control/trim_K23_R2.fastq \
        --min_contig_length 300 \
        --CPU 36 \
        --max_memory 100G \
        --output trinity_K23 \
        --full_cleanup
```  
    
    
So the useage information for Trinity program we use:
```
Usage:  Trinity [options]

Options (Required):
--seqType <string>       : type of reads: ('fa' or 'fq')
--max_memory <string>    : max memory to use by Trinity

if unpaired reads
--single <string>        : unpaired/single reads, one or more file names can be included

if paired reads
--left  <string>         :left reads, one or more file names (separated by commas, no spaces)
--right <string>         :right reads, one or more file names (separated by commas, no spaces) 

Options (optional)
--CPU <int>              : number of CPUs to use, default: 2
--min_contig_length <int>: minimum assembled contig length to report (def=200)
--output <string>        : directory for output
--full_cleanup           : only retain the Trinity fasta file, rename as ${output_dir}.Trinity.fasta
```  
   
     
The full slurm script is called [Trinity.sh](/Assembly/Trinity.sh), and can be found in the assembly directory.   

Trinity combines three independent software modules: Inchworm, Chrysalis, and Butterfly, applied sequentially to process large volumes of RNA-seq reads. Trinity partitions the sequence data into many individual de Bruijn graphs, each representing the transcriptional complexity at a given gene or locus, and then processes each graph independently to extract full-length splicing isoforms and to tease apart transcripts derived from paralogous genes. Briefly, the process works like so:   
    
_Inchworm_ assembles the RNA-seq data into the unique sequences of transcripts, often generating full-length transcripts for a dominant isoform, but then reports just the unique portions of alternatively spliced transcripts.   
   
_Chrysalis_ clusters the Inchworm contigs into clusters and constructs complete de Bruijn graphs for each cluster. Each cluster represents the full transcriptonal complexity for a given gene (or sets of genes that share sequences in common). Chrysalis then partitions the full read set among these disjoint graphs.   
    
_Butterfly_ then processes the individual graphs in parallel, tracing the paths that reads and pairs of reads take within the graph, ultimately reporting full-length transcripts for alternatively spliced isoforms, and teasing apart transcripts that corresponds to paralogous genes.   
   
During the **Trinity** run there will be lots of files will be grenerated. These checkpoint files will help us to restart from that specific point if for some reason the program stops for some other problems. Once the program ends sucessfully all these checkpoint files will be removed since we have requested a full cleanup using the `--full_cleanup` command. Clearing the files is very important as it will help us to remove all the unwanted files and also to keep the storage capacity and the number of files to a minimum. So at the end of a successful run we will end up with the following files:   
   
```
Assembly
├── trinity_K32.Trinity.fasta
├── trinity_K32.Trinity.fasta.gene_trans_map
├── trinity_U13.Trinity.fasta
├── trinity_U13.Trinity.fasta.gene_trans_map
├── trinity_U32.Trinity.fasta
└── trinity_U32.Trinity.fasta.gene_trans_map
```
   
So we will have three assembly files, one for each condition or time step.  
  
   
     
## 4. Determining and removing repeat modules

### Clustering using vsearch
Because we used RNA reads to sequence our transcriptome, chances are that there are multiples of the same reads varying slightly which create multiples of the same assembled sequence. Under this assumption, we may also assume that most of the modules in our assembled transcriptome are actually repeats, the results of the assembly of slightly different reads from the same gene. We want to remove the repeats of these modules to shorten the length of our transcriptome and make for more efficient work in the future. We can do this by partitioning and clustering the transcriptome, then taking only one module from each of the clusters. There is a very convenient software which performs all of this for us in the exact way just described: [vsearch](https://github.com/torognes/vsearch).

To obtain a set of unique genes from both runs, we will cluster the two resulting assemblies together. First, the two assembies will be combined into one file using the Unix command cat, which refers to concatanate.   

```bash
cat ../Assembly/trinity_U13.Trinity.fasta \
        ../Assembly/trinity_U32.Trinity.fasta \
        ../Assembly/trinity_K32.Trinity.fasta >> combine.fasta  
```  
   
Once the files are combined, we will use vsearch to find redundancy between the assembled transcripts and create a single output known as a centroids file. The threshold for clustering in this example is set to 80% identity.  
```bash
module load vsearch/2.4.3

vsearch --threads 32 --log LOGFile \
        --cluster_fast combine.fasta \
        --id 0.80 \
        --centroids centroids.fasta \
        --uc clusters.uc

```   

Command options in the vsearch program that we used:
```
Usage: vsearch [OPTIONS]
--threads INT               number of threads to use, zero for all cores (0)
--log FILENAME              write messages, timing and memory info to file
--cluster_fast FILENAME     cluster sequences after sorting by length
--id REAL                   reject if identity lower, accepted values: 0-1.0
--centroids FILENAME        output centroid sequences to FASTA file
--uc FILENAME               specify filename for UCLUST-like output
```   
  
The full script is called [vsearch.sh](/Clustering/vsearch.sh), which can be found in the **Clustering** folder. At the end of the run it will produce the following files:   
```
Clustering/
├── centroids.fasta
├── clusters.uc
├── combine.fasta
└── LOGFile
```     

The _centroids.fasta_ will contain the unique genes from the three asseblies.   
   
     
     
## 5. Identifying the Coding Regions   
   
### Identifying coding regions using TransDecoder   

Now that we have our reads assembled and clustered together into the single centroids file, we can use [TransDecoder](https://github.com/TransDecoder/TransDecoder/wiki) to determine optimal open reading frames from the assembly (ORFs). Assembled RNA-Seq transcripts may have 5′ or 3′ UTR sequence attached and this can make it difficult to determine the CDS in non-model species. We will not be going into how TransDecoder works. However, should you click the link you'll be happy to see that they have a very simple one paragraph explanation telling you exactly that.
Our first step is to determine all [open-reading-frames](https://en.wikipedia.org/wiki/Open_reading_frame). We can do this using the 'TransDecoder.LongOrfs' command. This command is quite simple, with one option, '-t', which is simply our centroid fasta! The command is therefore:   
   
```
module load TransDecoder/5.3.0

TransDecoder.LongOrfs -t ../clustering/centroids.fasta
```

The command useage would be:
```
Transdecoder.LongOrfs [options]

Required:
  -t <string>           transcripts.fasta
```


By default it will identify ORFs that are at least 100 amino acids long. (you can change this by using -m parameter). It will produce a folder called centroids.fasta.transdecoder_dir   

```
coding_regions
├── centroids.fasta.transdecoder_dir
│   ├── base_freqs.dat
│   ├── longest_orfs.cds
│   ├── longest_orfs.gff3
│   └── longest_orfs.pep
```


Next step is to, identify ORFs with homology to known proteins via blast or pfam searches. This will maximize the sensitivity for capturing the ORFs that have functional significance. We will be using the Pfram databases. Pfam stands for "Protein families", and is simply an absolutely massive database with mountains of searchable information on, well, you guessed it, protein families. We can scan the Pfam databases using the software hmmer, a database homologous-sequence fetcher. The Pfam databases are much too large to install on a local computer. However, you may find them on Xanadu in the directory '/isg/shared/databases/Pfam/Pfam-B.hmm', which is an hmmer file (must be an hmmer file for hmmer to scan!).  
   
```
hmmscan --cpu 16 \
        --domtblout pfam.domtblout \
        /isg/shared/databases/Pfam/Pfam-A.hmm \
        centroids.fasta.transdecoder_dir/longest_orfs.pep
```

Usage of the command:
```
Usage: hmmscan [-options] <hmmdb> <seqfile>

Options controlling output:
--domtblout <f>  : save parseable table of per-domain hits to file <f>

Other expert options:
--cpu <n>     : number of parallel CPU workers to use for multithreads  [2]
```


Once the run is completed it will create the following files in the directory.

```
coding_regions
├── pfam.domtblout
```

 
