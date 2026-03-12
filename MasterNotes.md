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
$ ls fastqc_out -- still on the compute node
