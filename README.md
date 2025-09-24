# About
Workflow and code used in [Pecoraro V. et al 2025]() manuscript

# Table of contents

- [Abstract](#Abstract)
- [Computational platform](#Computational-platform)
- [Indexing](#Indexing)
- [Read processing and pseudoalignment](#Read-processing-and-pseudoalignment)

# Abstract

# Computational platform
Genome indexing, read alignment and major parts of the processing for next generation sequencing data were done on [Euler HPC cluster](https://scicomp.ethz.ch/wiki/Euler) at ETH, Zurich.

# Indexing

## Prepare decoy aware file for Salmon

```bash

# Extract Genome fasta headers
grep "^>" <(gunzip -c Homo_sapiens.GRCh38.dna.primary_assembly.fa.gz) | cut -d " " -f 1 > decoys.txt

# Remove all ">"
sed -i.bak -e 's/>//g' decoys.txt

# Catenate transcriptome and Genome (order is important)
cat Homo_sapiens.GRCh38.cdna.all.fa.gz Homo_sapiens.GRCh38.dna.primary_assembly.fa.gz > Homo_sapiens_GRCh38_gentrome.fa.gz

```

## Indexing using salmon

Note: Salmon should be in PATH

```bash

# Load modules
module load stack/2024-06

# Directories
dir_in="/path/to/dir"
gen_in="/path/to/dir"

# Safety first
set -e -x


date +"%d-%m-%Y %T"
echo "-------------------------"
echo "Indexing"
echo "-------------------------"

# Index
salmon index \
--threads 12 \
--decoys "$gen_in"/decoys.txt \
--transcripts "$gen_in"/Homo_sapiens_GRCh38_gentrome.fa.gz \
--index "$gen_in"/salmon_index \
--kmerLen 31

date +"%d-%m-%Y %T"
echo "-------------------------"
echo "Indexing Done!"
echo "-------------------------"

```

# Read processing and pseudoalignment

```bash

# Load modules
module load stack/2024-06
module load python/3.11.6
module load pigz/2.7-oktqzxd
module load zlib/1.3-mktm5vz
module load bzip2/1.0.8-kxrmhba
module load boost/1.83.0
module load py-dnaio/0.10.0-l2umcaf
module load py-xopen/1.6.0-o55adyx
module load py-cutadapt/4.4-jfcyzb5


# Directories
dir_in="/path/to/dir"
gen_in="/path/to/dir"
THREADS=64  

# Safety first
set -e -x

# QC reads
for file in "$dir_in"/*.fastq.gz; do

    base=$(basename "$file" .fastq.gz)

    echo "-------------------------"
    echo "Cutadapt working on "${base}""
    echo "-------------------------"

    cutadapt \
    -j "$THREADS" \
    -q 25 \
    -m 25 \
    --cut 12 \
    -a AGATCGGAAGAGCACACGTCTGAACTCCAGTCA \
    -o "$dir_in"/${base}_qc.fastq.gz \
    "$dir_in"/${base}.fastq.gz \
    1> "$dir_in"/${base}_qc_cutadapt_log.txt

    echo "-------------------------"
    echo "Cutadapt done processing "${base}""
    echo "-------------------------"

done

# Quantifying
for file in "$dir_in"/*qc.fastq.gz; do

    base=$(basename "$file" _qc.fastq.gz)

    date +"%d-%m-%Y %T"
    echo "-------------------------"
    echo Quantifying "$base"
    echo "-------------------------"

    salmon quant \
    --threads "$THREADS" \
    --index "$gen_in"/salmon_index \
    --libType A \
    --gcBias \
    --seqBias \
    --useVBOpt \
    --validateMappings \
    --unmatedReads "$file" \
    --output "$dir_in"/${base}_quant

    date +"%d-%m-%Y %T"
    echo "-------------------------"
    echo Done with "$base"
    echo "-------------------------"

done

```