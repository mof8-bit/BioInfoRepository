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

3/17 --> assembling contigs

3/19 --> virsorter
ensure contigs are in megahit folder
create virosorter and votu folders
Install virosorter:

$ module load mamba
$ mamba create -y -n vs2-env -c conda-forge -c bioconda virsorter
$ rm -rf /home/mof8/group_proj/db/conda_envs
$ rm -rf /home/mof8/group_proj/db/.snakemake
$ virsorter setup -d /home/mof8/group_proj/db -j 4 --conda-frontend conda

Make slurm script (scripts folder) and submit. 
Script: #!/bin/bash
#SBATCH --job-name=virsorter
#SBATCH --nodes=1
#SBATCH --cpus-per-task=8                 
#SBATCH --mem=20G                         
#SBATCH --time=03:00:00     
#SBATCH --mail-type=END,FAIL
#SBATCH --mail-user=mof8@georgetown.edu              
#SBATCH --output=/home/mof8/group_proj/logs/virsorter.%j.out
#SBATCH --error=/home/mof8/group_proj/logs/virsorter.%j.err

# ==== Load mamba (students: no need to change) ====
module load mamba
source $(mamba info --base)/etc/profile.d/conda.sh

# Activate the environment where you had VirSorter2 installed
mamba activate vs2-env

# ==== Set paths and filenames (students: edit this block!) ====
#set up directories
INDIR=/home/mof8/group_proj/megahit      #directory where input will come from
OUTROOT=/home/mof8/group_proj/virsorter  #directory output will go
mkdir -p "${OUTROOT}"                                    #new directory to be created for outp$

SAMPLE_ID= final_contigs                                                 #just the basic sampl$
INPUT="${INDIR}/final.contigs.fa"                        #contig file name/location
OUTDIR="${OUTROOT}/vs2-${SAMPLE_ID}"                     #where you’ll find the outputs
mkdir -p "${OUTDIR}"


# ==== Run virsorter2 with >5kb cutoff and DNA virus categories first
echo "Running VirSorter2 on ${INPUT}"
virsorter run \
  -w "${OUTDIR}" \
  -i "${INPUT}" \
  --keep-original-seq \
  --include-groups dsDNAphage,NCLDV,ssDNA \
  --min-length 5000

echo "Done."

Find your results (in virsorter folder!)
Filter out anything smaller than 5 kb:

$ module load mamba/
$ mamba activate megahit-env  

Count contigs:

$ seqkit seq -m 5000 final-viral-combined.fa | grep -c “>”
$ seqkit seq -m 5000 final-viral-combined.fa > final-viral-combined_min5kb.fa

Cluster vOTUs:

vclust prefilter -i megahit/final.contigs.fa -o fltr.txt
vclust align -i final-viral-combined_min5kb.fa -o ani.tsv --filter fltr.txt
vclust cluster -i ani.tsv -o clusters.tsv --ids ani.ids.tsv --metric ani --ani 0.95 —-out-repr
vclust prefilter -i final-viral-combined_min5kb.fa -o fltr.txt

