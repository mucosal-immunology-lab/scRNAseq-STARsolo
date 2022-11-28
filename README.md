# Analysis of single-cell RNA sequencing data with STAR solo

Here we will describe the process of downloading publicly available scRNAseq data from the SRA database, and subsequent processing with STAR solo.

## Table of contents

- [Analysis of single-cell RNA sequencing data with STAR solo](#analysis-of-single-cell-rna-sequencing-data-with-star-solo)
  - [Table of contents](#table-of-contents)
  - [Installation of STAR](#installation-of-star)
    - [Download STAR](#download-star)
      - [Download the latest release and decompress it](#download-the-latest-release-and-decompress-it)
      - [Compile the tool with Linux](#compile-the-tool-with-linux)
      - [Add the tool to the PATH variable](#add-the-tool-to-the-path-variable)
    - [Download your genome](#download-your-genome)
    - [Download scRNAseq whitespace files](#download-scrnaseq-whitespace-files)
      - [BD Rhapsody whitespace files](#bd-rhapsody-whitespace-files)
    - [Generate genome index files](#generate-genome-index-files)
  - [Downloading sequencing reads](#downloading-sequencing-reads)
    - [Set up your SRA toolkit](#set-up-your-sra-toolkit)
    - [Retrieve the SRA files](#retrieve-the-sra-files)
    - [Convert the SRA accessions to FASTQ files](#convert-the-sra-accessions-to-fastq-files)
    - [Troubleshooting](#troubleshooting)
  - [Map reads to the genome](#map-reads-to-the-genome)
    - [BD Rhapsody](#bd-rhapsody)
    - [Drop-seq](#drop-seq)

## Installation of STAR

### Download STAR

Download and compile the latest version of [STAR](https://github.com/alexdobin/STAR). 

You can find the STAR manual [here](https://raw.githubusercontent.com/alexdobin/STAR/master/doc/STARmanual.pdf).

#### Download the latest release and decompress it

It is recommended that you download and install STAR on the project directory, in a dedicated tools folder or similar. The example below is given for version `2.7.10b`.

```bash
# Get latest STAR source from releases
wget https://github.com/alexdobin/STAR/archive/2.7.10b.tar.gz
tar -xzf 2.7.10b.tar.gz
```

#### Compile the tool with Linux

```bash
# Compile
cd STAR-2.7.10b/source
make STAR
```

#### Add the tool to the PATH variable

```bash
# Add a line to your .bashrc file
echo 'export PATH="$PATH:{path_to_toolsDB}/STAR-2.7.10b/source"' >> ~/.bashrc

# Export the PATH variable for this session
export PATH="$PATH:{path_to_toolsDB}/STAR-2.7.10b/source"
```

Ensure that you have added the tool to your `PATH` with the following command, which should return the designated directory. If not, you have not added the location to your `PATH` correctly.

```bash
which STAR
```

### Download your genome

Download the relevant species genome from GENCODE (recommended &ndash; [https://www.gencodegenes.org/](https://www.gencodegenes.org/)) or ENSEMBL (latest release &ndash; [http://ftp.ensembl.org/pub/release-108/](http://ftp.ensembl.org/pub/release-108/)).

You need to download both the GTF file and FASTA (DNA) file for your species. The most comprehensive annotations for your species is highly recommended.

An example using the human GRCh38.p13 GENCODE release 42:

```bash
# Download the GTF file
wget https://ftp.ebi.ac.uk/pub/databases/gencode/Gencode_human/release_42/gencode.v42.chr_patch_hapl_scaff.annotation.gtf.gz

# Download the FASTA file
wget https://ftp.ebi.ac.uk/pub/databases/gencode/Gencode_human/release_42/GRCh38.p13.genome.fa.gz

# Decompress everything
gunzip *
```

### Download scRNAseq whitespace files

For many scRNAseq systems, you are required to obtain whitespace files. For Drop-seq, this is not required, but an example of the process for the BD Rhapsody is explained below.

#### BD Rhapsody whitespace files

The structure of the BD Rhapsody is explained [here](https://teichlab.github.io/scg_lib_structs/methods_html/BD_Rhapsody.html).

STARsolo requires all 3 BD Rhapsody whitespace files.

```bash
# Navigate to a database folder - perhaps on the project drive
cd path/to/db
wget https://teichlab.github.io/scg_lib_structs/data/BD_CLS1.txt
wget https://teichlab.github.io/scg_lib_structs/data/BD_CLS2.txt
wget https://teichlab.github.io/scg_lib_structs/data/BD_CLS3.txt
```

### Generate genome index files

This process is explained in greater detail in the [STAR manual](https://raw.githubusercontent.com/alexdobin/STAR/master/doc/STARmanual.pdf).

The basic options to generate your genome index are as follows:

```bash
--runThreadN NumberOfThreads
--runMode genomeGenerate
--genomeDir /path/to/genomeDir
--genomeFastaFiles /path/to/genome/fasta1 /path/to/genome/fasta2 ...
--sjdbGTFfile /path/to/annotations.gtf
--sjdbOverhang ReadLength-1
```

You will need to create the genome directory (`genomeDir`) prior to generating the genome index. Ensure that `STAR` is in your `PATH` variable (as shown [above](#add-the-tool-to-the-path-variable)).

An example of this process is as follows:

```bash
#!/bin/bash

genomeDir="{path_to_DB}/gencode/human"

STAR \
  --runThreadN 20 \
  --genomeDir "${genomeDir}" \
  --runMode genomeGenerate \
  --genomeFastaFiles "${genomeDir}/GRCh38.p13.genome.fa" \
  --sjdbGTFfile "${genomeDir}/gencode.v42.chr_patch_hapl_scaff.annotation.gtf" \
  --sjdbOverhang 99
```

## Downloading sequencing reads

### Set up your SRA toolkit

Firstly, we need to set up the SRA toolkit. Full instructions for this can be found [here](https://github.com/ncbi/sra-tools/wiki). Follow their instructions, and ensure everything is working as expected &ndash; they provide a quick check for this by downloading the first two lines of a FASTQ file.

You will require [cloud credentials](https://github.com/ncbi/sra-tools/wiki/04.-Cloud-Credentials). Try the Google credentials first, and save the JSON credentials file somewhere, as you will need to provide this file in the `vdb-config -i` step &ndash; this will work either locally or on the cluster. All other steps work as described.

### Retrieve the SRA files

The first step of actually acquiring the sequencing runs is to `prefetch` the accessions. Essentially, a combination of `prefetch` + `fasterq-dump` is the fastest way to extract FASTQ files from SRA accessions. Even better, the `prefetch` tool can be invoked multiple times in the event the download is not successful, and it will pick up from where it left off.

Something to note here:

* The `prefetch` tool downloads to a directory named by accession. E.g. `prefetch SRR000001` will create a directory named `SRR000001` in the current directory. Make sure that if you move the `SRR000001` directory, you don't rename it as the conversion-tool will need to find the original directory.

To retrieve multiple SRA accessions, you should have a `.txt` file containing a single accession per line, and you can run the following `bash` command to download them to some parent directory (`path/to/raw_data`).

```bash
prefetch --progress --output-directory path/to/raw_data --option-file SRA_accession_list.txt
```

As described above, you will now have a number of folders inside the parent output folder you designated; one folder for each SRA.

### Convert the SRA accessions to FASTQ files

First we need to create two bash scripts: 1) a slurm submission script that will submit a single job, and 2) a bash script that loops through job submissions.

Because of the limitations on job submission for the genomics cluster, we will first need to split the SRA accession list into groups of 32. This will limit usage to the maximum of 4 nodes that a user can occupy at any one time. I.e. 6GB x 6 CPUs = 36GB RAM &rarr; 36GB x 8 = 288GB and 6 CPUS x 8 = 48 CPUs &rarr; Each genomics node has 334GB RAM and 48 CPUs. Therefore we can run 8 jobs per node, and 4 x 8 = 32 jobs maximum.

This can be done manually, or via the following code. It will create a number of files with a suffix in alphabetical order. Of course if you have less than this number of jobs, you can skip this step.

```bash
split -l 32 full_accession_list.txt fasterq-dump- -a 1 --additional-suffix .txt
```

An example of the split file would be (with filename `fq-dump-job-a`):

```bash
SRR8284756
SRR8284757
SRR8284758
SRR8284759
SRR8284760
SRR8284761
SRR8284762
SRR8284763
... etc.
```

The slurm submission script is as follows, and is named `submit_fasterq_dump.sh`.

```bash
#!/bin/bash
#SBATCH --job-name=fasterq-dump
#SBATCH --account=mf33
#SBATCH --time=4:00:00
#SBATCH --mem-per-cpu=6G
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=6
#SBATCH --partition=genomics
#SBATCH --qos=genomics

fasterq-dump $1 --split-files \
  --include-technical \
  --outdir ../raw_fastq \
  --verbose \
  --progress
```

Finally, our bash script to loop through job submissions is as follows, and is named `convert_fastq.sh`.

```bash
#!/bin/bash
while read SRA; do
  FILE="../raw_data/${SRA}/"
  sbatch submit_fasterq_dump.sh "$FILE"
  sleep 0.5
done < $1
```

Therefore, to run this set of (<= 32) `fasterq-dump` jobs, we run the command below. We can then continue with the others afterwards. It does take longer, and require user input to run each group of jobs, however it respects everyone else's ability to also use the genomics partition.

```bash
bash convert_fastq.sh fq-dump-job-a

OR

bash convert_fastq.sh full_accession_list.txt if <= 32 SRAs
```

### Troubleshooting

If you get an error such as below, then it is likely you trimmed your SRA accession list from a text file that originated on a Windows system.

```bash
Preference setting is: Prefer SRA Normalized Format files with full base quality scores if available.
2022-11-28T04:38:05 fasterq-dump.3.0.1 err: error unexpected while resolving query within virtual file system module - No accession to process ( 500 )
Failed to call external services.
```

If you view your file via `cat -v filename`, and you see the `^M` new line marker at the ends, you can fix this using the following command to reset the new line markers for use with Unix systems.

```bash
sed -i -e "s/\r//g" filename
```

## Map reads to the genome

This process will be slightly different depending on the scRNAseq system you used, but a couple of example scripts are given below.

### BD Rhapsody

`STARsolo_map_BDRhapsody.sh`

```bash
#!/bin/bash

genomeDir="/path/to/genomeDir"
seqDir="/path/to/sequencing/files"

/home/{path}/STAR-2.7.10b/source/STAR \
        --outFileNamePrefix "${seqDir}/sample1" \
        --genomeDir "${genomeDir}/STARgenomeIndex" \
        --runThreadN 20 \
        --readFilesIn "${seqDir}/data/sample1_R2.fastq.gz" "${seqDir}/data/sample1_R1.fastq.gz" \
        --readFilesCommand zcat \
        --sjdbGTFfile "${genomeDir}/db/gencode.vM27.chr_patch_hapl_scaff.annotation.gtf" \
        --soloType CB_UMI_Complex \
        --soloCBmatchWLtype 1MM \
        --soloCBwhitelist "${genomeDir}/BD_CLS1.txt" "${genomeDir}/BD_CLS2.txt" "${genomeDir}/BD_CLS3.txt" \
        --soloUMIlen 8 \
        --soloCBposition 0_0_0_8 0_21_0_29 0_43_0_51 \
        --soloUMIposition 0_52_0_59 \
        --soloCellFilter EmptyDrops_CR 3000 0.99 10 45000 90000 500 0.01 20000 0.01 10000
```

### Drop-seq

`STARsolo_map_dropSeq.sh`

```bash
#!/bin/bash

genomeDir="/path/to/genomeDir"
seqDir="/path/to/sequencing/files"

/home/{path}/STAR-2.7.10b/source/STAR \
        --outFileNamePrefix "..." \
        --genomeDir "${genomeDir}/STARgenomeIndex" \
        --runThreadN 19 \
        --readFilesIn "${seqDir}/sample1_R2_001.fastq.gz" "${seqDir}/sample1_R1_001.fastq.gz" \
        --readFilesCommand zcat \
        --sjdbGTFfile "${genomeDir}/db/gencode.vM27.chr_patch_hapl_scaff.annotation.gtf" \
        --soloType Droplet
        --soloCBwhitelist none  \
        --soloCBstart 1 \
        --soloCBlen 12\
        --soloUMIstart 13 \
        --soloUMIlen 8
```