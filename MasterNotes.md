Go to home directory. Load anaconda module. Install SRA-tools through an environment: 
$ module load anaconda3
$ conda create -n sra_env -c bioconda sra-tools
→ say yes
$ conda activate sra_env
Mkdir for fastq. Cd into fastq directory. 
$ prefetch [accession number]
Cd into new downloaded directory. Change files into fastq format, then compress.
$ fasterq-dump *.sra
$ gzip *.fastq

