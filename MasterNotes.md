Get data

Go to home directory. Load anaconda module. Install SRA-tools through an environment: 
$ module load anaconda3 --  load anaconda module

$ conda create -n sra_env -c bioconda sra-tools -- create SRA tools enviornment 
→ say yes

$ conda init -- initiate anaconda -- initialize conda and reload shell
$ source ~/.bashrc -- reload shell

$ conda activate sra_env -- activate enviornment

$ cd /home/mof8/group_proj/fastqc_raw

$ prefetch [SAMN08784154] -- downloaded sequencing data from SRA

$ mkdir fastqc_raw -- make directory for the raw fastqc data 
$ cd fastqc_raw -- move to that directory 

$ fasterq-dump *.sra -- convert SRA file into FASTQ

$ gzip *.fastq -- compress fastq files
get organized -- name 

Fastqc

Fastqc files to focus on: SRR37587572_1.fastq SRR37587572_2.fastq

$ srun --pty bash -- enter interactive mode on a compute node 
$ module load fastqc -- load FastQC
$ fastqc -h -- confirm FastQC is available
$ fastqc -o fastqc_out SRR37587572_1.fastq SRR37587572_2.fastq -- run fastq
$ fastqc -o fastqc_out sample_R1.fastq.gz sample_R2.fastq.gz -- example with two paired-end files
$ ls fastqc_out -- still on the compute node -- we should see files like SRR37587572_1.fastqc.html 

Trimmomatic

$ SLURM script: 

#!/bin/bash
#SBATCH --job-name=projecttrims.SBATCH --output=z01.%x
#SBATCH --mail-type=END,FAIL --mail-user=sja111@georgetown.edu
#SBATCH --nodes=1 --ntasks=1 --cpus-per-task1 --time=3:00:00

shopt -s expand_aliases
module load trimmomatic

adapters=/home/sja111/fastq_files_project/TruSeq-PE.fa
input_R1=/home/sja111/fastq_files_project/raw/SRR37587572_1.fastqc.gz
input_R2=/home/sja111/fastq_files_project/raw/SRR37587572_2.fastqc.gz
output_R1_PE=/home/sja111/fastq_files_project/trimmed/SRR37587572_1_trmPE.fq.gz
output_R1_SE=/home/sja111/fastq_files_project/trimmed/SRR37587572_1_trmPE.fq.gz
output_R2_PE=/home/sja111/fastq_files_project/trimmed/SRR37587572_2_trmPE.fq.gz
output_R2_SE=/home/sja111/fastq_files_project/trimmed/SRR37587572_2_trmPE.fq.gz

trimmomatic PE \
$input_R1 \ 
$input_R2 \
$output_R1_PE $output_R1_SE \
$output_R2_PE $output_R2_SE \

LEADING:10
TRAILING:10
ILLUMINACLIP:$adapters:2:30:10 \
SLIDINGWINDOW:4:15 \
MINLEN:75 \

$ info about surviving reads: 
$ anything noteworthy/quality: 
