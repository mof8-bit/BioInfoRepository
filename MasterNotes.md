# Metagenome Project: Maeve & Salvatore
### notes to self: include overall goal of project up here? also include what folders we created. 

## Class 15: Intro to GitHub and Installing R
1. Create group GitHub repository with ReadMe file to keep track of our workflow (makes it reproducible)
2. Install R and RStudio (for analysis at the end of the project)

## Class 16: Get Organized, Download Sample Data, and Start Workflow
### Step 1: Download your FASTQ files
1. Get data --> sample 7: bog frozen replicate. SAMN08784154.
2. Go to home directory 
3. Load anaconda module on the HPC so we can use conda
```bash
module load anaconda3
```
4. Create a conda enviornment and install the STA Toolkit in it 

```bash
conda create -n sra_env -c bioconda sra-tools 
# say yes
```
5. Activate the environment so SRA Toolkit commands will work
```bash
conda activate sra_env
```
6. Make and enter fastqc_raw directory (folder where you want your sequencing files downloaded)  
```bash
mkdir fastqc_raw
cd /home/mof8/group_proj/fastqc_raw
```
7. Download sequencing data from NCBI SRA
```bash
prefetch [SAMN08784154]
```
8. Convert .sra file into FASTQ format
```bash
fasterq-dump *.sra
``` 
9. Compress FASTQ file
```bash
gzip *.fastq
```
10. Fastqc files to focus on: SRR37587572_1.fastq SRR37587572_2.fastq --> we got a lot of files and chose these two to focus on

### Step 2: FASTQC of Raw Reads

1. Enter interactive mode on a compute node
```bash
srun --pty bash
```
2. Confirm the prompt changes to make sure your commandd line knows you are on the compute node
3. Load the FastQC software so you can use the fastqc command
```bash
module load fastqc
```
4. Show the help menu for FastQC to confirm that it loaded correctly and to see available options
```bash
fastqc -h
```
5. Make a folder called fastqc_out for the FastQC results
```bash
mkdir -p fastqc_out
```
6. Run FastQC on paired end reads
```bash
fastqc -o fastqc_out SRR37587572_1.fastq SRR37587572_2.fastq
```
fastqc is the program
-o fastqc_out puts the output files into the fastqc_out folder
SRR37587572_1.fastq SRR37587572_2.fastq are the established files of interest

7. List the files inside the output folder so you can confirm that FastQC finished and created resutls
```bash
ls fastqc_out
```
We should see files like SRR37587572_1.fastqc.html and .zip
.html -- the report you open and read
.zip thes same results in a compressed folder

8. Download the .html report to your computer

### Step 3: Trimmomatic and FastQC of Trimmed
1. Load trimmommatic
```bash
module load trimmomatic
```
2. Run paired-end trimming using this slurm script: 
```bash
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

## *** we need to insert the trimming statistics here ***

3. FastQC of trimmed reads
```bash
fastqc -o projectfastqc_out \
trimmed/SRR37587572_1_paired.fastq.gz \
trimmed/SRR37587572_2_paired.fastq.gz
```
## *** we evaluation of trimmed output files here ***

## Class 17: Assembly of Contigs

### Goal
The goal of this step was to assemble our cleaned paired-end sequencing reads into longer contiguous sequences, called **contigs**, using **MEGAHIT**. This is important because downstream viral analysis tools work on assembled contigs rather than short raw reads.

---

### Step 1: Get organized

Before running assembly, we made sure that:

1. Our reads had already been cleaned using **FastQC** and **Trimmomatic**
2. Our cleaned files were paired-end reads with clear names
3. We knew the full path to the cleaned read files
4. We had an output directory ready for the MEGAHIT results

Example cleaned read files:

```bash
SRR37587572_1_paired.fastq.gz
SRR37587572_2_paired.fastq.gz
```
### Step 2: Install MEGAHIT 
We used mamba to create an enviornment containing MEGAHIT. It assembles short sequencing reads into longer contigs by finding overlaps between reads. This gives us larger sequences that are much more useful for downstream viral identification. 

```bash
module load mamba/
mamba create -y -n megahit-env -c conda-forge -c bioconda megahit
```
module load mamba/ --> loads mamba onto the HPC
mamba create -->  creates a new enviornment
-y --> automatically says yes to installation promps
-n megahit-env names the enviornment megahit-env
-c conda-forge -c bioconda --> tels mamba which channels to install from
megahit --> is the software package being installed

### Step 3: Write SLURM script to run MEGAHIT

1. Use this slurm script
```bash
#!/bin/bash
#SBATCH --job-name=megahit_sample
#SBATCH --nodes=1
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G
#SBATCH --time=03:00:00
#SBATCH --output=/home/mof8/group_proj/logs/megahit_%j.out
#SBATCH --error=/home/mof8/group_proj/logs/megahit_%j.err
#SBATCH --mail-type=END,FAIL
#SBATCH --mail-user=YOUR_EMAIL@georgetown.edu

module load mamba/
source $(mamba info --base)/etc/profile.d/conda.sh
conda activate megahit-env

READ1=/home/mof8/group_proj/trimmed/SRR37587572_1_paired.fastq.gz
READ2=/home/mof8/group_proj/trimmed/SRR37587572_2_paired.fastq.gz
OUTDIR=/home/mof8/group_proj/megahit/SRR37587572_megahit_out

megahit \
  -1 ${READ1} \
  -2 ${READ2} \
  -t ${SLURM_CPUS_PER_TASK} \
  -o ${OUTDIR}

echo "Done. Contigs should be in ${OUTDIR}/final.contigs.fa"
```
2. Submit job using
```bash
sbatch megahit.sbatch
```
### Step 4: Find the Results
1. Navigate to output directory after job is finished and look at assembly results
-- most important file is final.contigs.fa
2. Inspect the output using
```bash
cd /home/mof8/group_proj/megahit/SRR37587572_megahit_out
ls
head final.contigs.fa
grep -c "^>" final.contigs.fa
```
## *** what did we see? ***

### Step 5: Check Assembly Quality with seqkit 
1. Install seqkit
```bash
module load mamba/
source $(mamba info --base)/etc/profile.d/conda.sh
conda activate megahit-env
mamba install -c bioconda seqkit
```
2. Find assembly statistics like number of contigs, total length of contigs, min/max/avg contig length, and N50
```bash
seqkit stats -a final.contigs.fa
```
## *paste seqkit output here*

## Class 18
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

Mar 24
Set up checkv in newly made checkv folder
module load checkv						#its available as a module on the HPC
checkv download_database ./				#make sure you’re in your checkv folder!
make slurm script
look at quality_summary_votus.tsv: We noticed that most of the vOTUs were lower quality. In fact, at first glance we cannot see any that are not lower quality. 
