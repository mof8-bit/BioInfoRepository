# Metagenome Project: Maeve & Salvatore
**notes to self: include overall goal of project up here? also include what folders we created.** 

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
**evaluation of trimmed output files here** 

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
**what did we see?**

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
**paste seqkit output here**

## Class 18: VirSorter2 and Clustering into vOTUs

### Goal
The goal of this class was to identify likely viral contigs from our assembled metagenome and then begin clustering those viral contigs into **vOTUs** (viral operational taxonomic units). We first used **VirSorter2** to predict which contigs were viral, then filtered those viral contigs by length, and finally used **vclust** to cluster similar viral sequences based on average nucleotide identity (ANI).

### Step 1: Get organized

Before starting VirSorter2, we made sure that:

1. The correct contig file from the MEGAHIT assembly was in the megahit output folder
2. I knew the full path to the contig file
3. I had directories prepared for the VirSorter2 output and later vOTU clustering output

The  input file for this step was the assembled contig file from MEGAHIT:

```bash
/home/mof8/group_proj/megahit/final.contigs.fa
```
### Step 2: Install VirSorter2
```bash
module load mamba
mamba create -y -n vs2-env -c conda-forge -c bioconda virsorter
```
module load mamba --> load mamba onto hpc
mamba create --> creates a new enviornment
-y --> automatically says yes to installation prompts
-n vs2-env --> names the enviornment vs2-env
-c conda-forge -c bioconda --> tells mamba where to install the package from
virsorter is the package being installed

### Step 3: Download the VirSorter2 databases
```bash
module load mamba 
source $(mamba info --base)/etc/profile.d/conda.sh
mamba activate vs2-env # activates VirSorter2 enviornment

rm -rf /home/mof8/group_proj/db/conda_envs # removes a failed conda_envs directory from previous attempt
rm -rf /home/mof8/group_proj/db/.snakemake # removes failed snakemake directory from a previous attempt

virsorter setup -d /home/mof8/group_proj/db -j 4 --conda-frontend conda # downloads and prepares the virsorter2 databases

```
### Step 4: Write and Submit a SLURM Script for VirSorter2
1. Write script
```bash
Script: #!/bin/bash
#SBATCH --job-name=virsorter
#SBATCH --nodes=1
#SBATCH --cpus-per-task=8                 
#SBATCH --mem=20G                         
#SBATCH --time=03:00:00     
#SBATCH --mail-type=END,FAIL
#SBATCH --mail-user=sja111@georgetown.edu              
#SBATCH --output=/home/sja111/Bioinfo_project/Virsorter/logs/virsorter.%j.out
#SBATCH --error=/home/sja111/Bioinfo_project/Virsorter/logs/virsorter.%j.err

# ==== Load mamba (students: no need to change) ====
module load mamba

# Activate the environment where you had VirSorter2 installed
mamba activate vs2-env

# ==== Set paths and filenames (students: edit this block!) ====
#set up directories
INDIR=/home/sja111/Bioinfo_project/Virsorter/input
OUTROOT=/home/sja111/Bioinfo_project/Virsorter
mkdir -p "${OUTROOT}"

SAMPLE_ID=VirsortSample
INPUT="${INDIR}/dori_luby_final.contigs.fa"
OUTDIR="${OUTROOT}/vs2-${SAMPLE_ID}"
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
```
Note that dori_luby_final.contigs.fa is a file from another group. These were used because our contigs were of insufficient length.

2. Submit
```bash
sbatch virsorter.sbatch
```
### Alernative Directions? (pasted in by prof)
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

## Class 19: CheckV and Alignment to vOTUs

### Goal
The goal of this class was to evaluate the quality of our viral sequences using **CheckV**, then prepare for analysis by aligning our cleaned reads to the class **vOTUs** reference with **Bowtie2** and **Samtools**. 

### Step 1: Set up CheckV

CheckV is used after VirSorter2 and clustering because viral contigs can still be partial or contaminated with host DNA. CheckV estimates completeness, identifies host contamination, trims host-derived regions, and assigns quality categories such as complete, high, medium, low, or undetermined. 

Before running CheckV, we made sure we had:
1. Our vOTU FASTA file
2. Its full path
3. A `checkv` directory for the database and output files

Download checkv in checkv folder

```bash
module load checkv # its available as a module on the HPC
checkv download_database ./ # make sure you’re in your checkv folder!
```
### Step 2: Write and Submit SLURM script for CheckV 
1. Write SLURM script
```bash
#!/bin/bash
#SBATCH --job-name=checkv
#SBATCH --output=/home/mof8/group_proj/logs/checkv-%j.out
#SBATCH --error=/home/mof8/group_proj/logs/checkv-%j.err
#SBATCH --time=03:00:00
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=16
#SBATCH --mem=16G
#SBATCH --mail-type=END,FAIL
#SBATCH --mail-user=mof8@georgetown.edu

# ==== Load checkv module ====
module load checkv

# ==== Set variables, paths, and filenames ====
CHECKVDB="/home/mof8/group_proj/checkv/checkv-db-v1.5"
SAMPLE_ID="vOTUs"
INPUT="/home/mof8/group_proj/votus/votus_10kb_6samples.fna"
OUTDIR="/home/mof8/group_proj/checkv/${SAMPLE_ID}"

mkdir -p "${OUTDIR}"

# ==== Run CheckV ====
echo "Running CheckV on ${INPUT}"
checkv end_to_end "${INPUT}" "${OUTDIR}" -d "${CHECKVDB}" -t ${SLURM_CPUS_PER_TASK}
echo "Done."
```
2. Submit using
```bash
sbatch checkv.sbatch
```
### Step 3: Find and Interpret results
1. Look at output files-- the main one is quality_summary.tsv
2. Inspect output
```bash
cd /home/mof8/group_proj/checkv/vOTUs
ls
head quality_summary.tsv
```   
### Step 4: Download the pooled class vOTUs
```bash
gcloud storage cp gs://gu-biology-dept-class/ClassProject/votus_10kb_6samples.fna /home/mof8/group_proj/bowtie2/
```
### Step 5: Set up Bowtie2 directory and build the index
```bash
srun --pty bash # starts an interactive compute session
module load bowtie2 # loads bowtie2
bowtie2-build votus_10kb_6samples.fna votu_index # creates the bowtie2 index files from the pooled vOTUs FASTA
exit
```
### *** revisit my scripts for these *** Step 6: Write and Submit a Bowtie2 and Samtools SLURM script
1. Scripts
2. Submit the job
```bash
sbatch bowtie2.sbatch
```
### Step 7: Upload final Bowtie2 output files to class bucket 
```bash
gcloud storage cp sample4_mof8_sorted.bam gs://gu-biology-dept-class/ClassProject/bam
```
**you forgot to rename this sample4_mof8... that made it hard to identify which sample you added to bucket! (maeve)** 

## Class 20: Make Figures to Look at Ecology
Review: 
1. We built an index of the reference genome using bowtie-build-- this lets Bowtie2 quickly find candidate locations in the genome where a short read could align without scanning the genome linearly for every read
2. We aligned out FASTQ reads to the index with bowtie2, which searches for where in the referecne genome each read fits best while allowing small indels/mismatches and write alignments to a SAM file
3. We converted the SAM file to a BAM file which is easier to process
4. She took our sorted.bam files and used CoverM to calculate how many reads mapped and how well it is covered for each vOTU

Today:
1. Download RStudio
2. Make a local working directory on your computer
3. Download .tsv and .R files from bucket /ClassProject/
4. Make either heatmap of abundance or richness tables

**what do we see**? 
