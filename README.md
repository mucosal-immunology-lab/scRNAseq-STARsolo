# Analysis of single-cell RNA sequencing data with STAR solo

Here we will describe the process of downloading publicly available scRNAseq data from the SRA database, and subsequent processing with STAR solo.

## Table of contents

- [Analysis of single-cell RNA sequencing data with STAR solo](#analysis-of-single-cell-rna-sequencing-data-with-star-solo)
  - [Table of contents](#table-of-contents)
  - [Downloading sequencing reads](#downloading-sequencing-reads)
    - [Set up your SRA toolkit](#set-up-your-sra-toolkit)
    - [Retrieve the SRA files](#retrieve-the-sra-files)
    - [Convert the SRA accessions to FASTQ files](#convert-the-sra-accessions-to-fastq-files)


## Downloading sequencing reads

### Set up your SRA toolkit

Firstly, we need to set up the SRA toolkit. Full instructions for this can be found [here](https://github.com/ncbi/sra-tools/wiki). Follow their instructions, and ensure everything is working as expected &ndash; they provide a quick check for this by downloading the first two lines of a FASTQ file.

You will require [cloud credentials](https://github.com/ncbi/sra-tools/wiki/04.-Cloud-Credentials). Try the Google credentials first, and save the JSON credentials file somewhere, as you will need to provide this file in the `vdb-config -i` step &ndash; this will work either locally or on the cluster. All other steps work as described.

### Retrieve the SRA files

The first step of actually acquiring the sequencing runs is to `prefetch` the accessions. Essentially, a combination of `prefetch` + `fasterq-dump` is the fastest way to extract FASTQ files from SRA accessions. Even better, the `prefetch` tool can be invoked multiple times in the event the download is not successful, and it will pick up from where it left off.

Something to note here:

* The prefetch-tool downloads to a directory named by accession. E.g. `prefetch SRR000001` will create a directory named `SRR000001` in the current directory. Make sure that if you move the `SRR000001` directory, you don't rename it as the conversion-tool will need to find the original directory.

To retrieve multiple SRA accessions, you should have a `.txt` file containing a single accession per line, and you can run the following `bash` command to download them to some parent directory (`path/to/raw_data`).

```bash
prefetch --progress --output-directory path/to/raw_data --option-file SRA_accession_list.txt
```

As described above, you will now have a number of folders inside the parent output folder you designated; one folder for each SRA.

### Convert the SRA accessions to FASTQ files