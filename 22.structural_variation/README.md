## Structural variant calling ##

This tutorial steps through some basic tasks in alignment and structural variant calling using some maize data sets.
This tutorial is modified from the [Krumlov_teaching_SV](https://github.com/fritzsedlazeck/teaching_material/blob/main/Krumlov/Krumlov_teaching_SV.md).

Sequences used in this tutorial were derived from:  
1. SRR15447420: [Chen et al. (2023)](https://www.nature.com/articles/s41588-023-01419-6)  
2. SRR11606869: [Hon et al. (2020)](https://www.nature.com/articles/s41597-020-00743-4)  
3. Mo17 assembly: [Chen et al. (2023)](https://www.nature.com/articles/s41588-023-01419-6)  
4. B73 assembly: [Hufford et al. (2021)](https://www.science.org/doi/10.1126/science.abg5289)  

#### Goals of this module ####
The goal of this module is to get you familiarized with structural variant identification across assembly, short read based mapping and long read based mapping.
For Structural Variation (SV) detection we will be using multiple methods (Assemblytics, minimap2, Sniffles) and then compare them to obtain more insights about the advantages and disadvantages of the different approaches.

All the instructions below require you to think along. You are responsible for a clear data structure in your workspace and for finding the individual files that you need. I am happy to assist, but please take a moment to look around (`ls -l`) or think where a certain file could be.

The main steps in this module are:
1. Short read based InDel detection (using the variants from [21.variant_calling](../21.variant_calling))
2. Assembly based SV detection (using [Assemblytics](https://github.com/MariaNattestad/Assemblytics))
3. Long read based SV detection (using [Sniffles](https://github.com/fritzsedlazeck/Sniffles))
4. SV comparison (using [SURVIVOR](https://github.com/fritzsedlazeck/SURVIVOR))

### Set up ###
#### 1. Request compute resources ####
The `-c 48` requests a server with 48 CPU cores. `-t 1:00:00` requests 1 hour of time, modify it if you need longer time.
```
sinteractive -A PAS3124 -c 48 -t 1:00:00
```

#### 2. Update your course folder ####
Replace `name_data` with your folder name in all commands below. If you don't have the course folder yet, set it up first (step 2 in [Helixer.md](../20.gene_annotation/Helixer.md)).
```
cd /fs/scratch/PAS3124/name_data/MG5720
# sync to the latest course files; your data and outputs are kept
git fetch origin
git reset origin/main
git restore .
```

#### 3. Link the sequences and the lecture 21 variants ####
```
cd /fs/scratch/PAS3124/name_data/MG5720/22.structural_variation
# B73 and Mo17 assemblies of the same 5 Mb region
ln -s ../data/B73_chr2_5M.fa
ln -s ../data/Mo17_chr2_5M.fa
# PacBio HiFi reads of B73 and Mo17 from this region
ln -s ../data/B73_chr2_5M.fastq.gz
ln -s ../data/Mo17_chr2_5M.fastq.gz
# the annotated short-read variants from lecture 21
ln -s ../data/Zea.chr2_5M.filtered.annotated.vcf
ls -l
```

#### 4. Create the "sv" environment ####
We use a new environment called "sv", because the packages used in today's analysis conflict with the "mapping" environment. `sv.yml` lists the tools at versions tested for this class.
```
cd /fs/scratch/PAS3124/name_data/MG5720/22.structural_variation
module load miniconda3/24.1.2-py310

# install the tools, use ONE of the two options
# option 1 (recommended): create the "sv" environment from sv.yml (about 2 minutes)
conda env create -f sv.yml -y
# option 2: created "sv" with "conda create -n sv ..."
conda create -n sv -c bioconda -c conda-forge survivor "sniffles>2" assemblytics minimap2 bcftools samtools bedtools mummer4 -y

conda activate sv

# test if the tools are installed: each line prints a version or usage, not "command not found"
nucmer --version
assemblytics -h | head -n 1
sniffles --version
samtools --version | head -n 1
```

#### If the installation was interrupted ####
If the installation is interrupted, or a command in step 4 failed, request a new session (step 1) and run step 4 with option 1. `-y` replaces the old `sv` environment, including a half-installed one.

### Short read based InDel detection ###
Let's first look at the small Insertions and Deletions (InDels) detected through aligning Illumina short reads. These InDels are usually shorter than 50 bp due to read length limitations and alignment uncertainties.

Identify short InDels from the short-read-based VCF that was created in the SNP lecture.
```
cd /fs/scratch/PAS3124/name_data/MG5720/22.structural_variation
module load miniconda3/24.1.2-py310
conda activate sv
bcftools view -v indels Zea.chr2_5M.filtered.annotated.vcf > B73_chr2_5M.Illumina.InDel.vcf

# check it out
less -S B73_chr2_5M.Illumina.InDel.vcf

# how many InDels could we identify?
grep -vc '#' B73_chr2_5M.Illumina.InDel.vcf
```

### Assembly based SV detection ###
As discussed in the lecture, assembly based SV detection is quite comprehensive. Nevertheless, before we start, we need to align a new assembly to an existing e.g. reference genome to identify structural differences. For this we will be using MUMmer (`nucmer`). Alternatively, you could also utilize `Dipcall` as a program especially if you have a phased assembly at hand. MUMmer is a commonly used package that allows you to rapidly compare two sequences together and includes multiple packages for summary reports over the aligned sequences. This includes variant reporting, coordinate reporting or even the generation of dot plots. As shown in the lecture, a dot plot is a very helpful and intuitive method to compare sequences.

To begin to call variants, we need to first compare the reference to our assembly. Now initiate the alignment. It will create an alignment file named `out.delta` for our next step.
```
nucmer --maxmatch -l 100 -c 500 B73_chr2_5M.fa Mo17_chr2_5M.fa
```

Use Assemblytics to identify SV candidates with a unique anchor length of 100 bp, a minimum variant size of 50 bp and a maximum variant size of 50 kb. The results go to the `B73-Mo17_chr2_5M` folder.
```
assemblytics -d out.delta -o B73-Mo17_chr2_5M -l 100 -min 50 -max 50000
```

Let's check out the summary of SVs detected by Assemblytics:
```
less B73-Mo17_chr2_5M/assemblytics_structural_variants.bed
less B73-Mo17_chr2_5M/assemblytics_structural_variants_summary.txt
```

To convert the Assemblytics SVs into VCF format, we will need SURVIVOR. The `10` keeps variants of at least 10 bp, so all the Assemblytics SVs (50 bp or longer) are kept.
```
SURVIVOR convertAssemblytics B73-Mo17_chr2_5M/assemblytics_structural_variants.bed 10 B73_chr2_5M.WGA.SV.vcf
less -S B73_chr2_5M.WGA.SV.vcf
```

Calculate the average length of SVs
```
# SV length in the Assemblytics output
grep -v '#' B73-Mo17_chr2_5M/assemblytics_structural_variants.bed | awk '{LEN += $5} END {print LEN/NR}'

# SV length in the converted VCF
grep -v '#' B73_chr2_5M.WGA.SV.vcf | sed 's/.*SVLEN=//; s/;PE.*//' | awk '{LEN+=$0} END {print LEN/NR}'
```

### Long read based SV detection ###
Finally, we are ready for detecting SVs using PacBio long reads. First, let's align the long reads to the reference (less than a minute).
```
cd /fs/scratch/PAS3124/name_data/MG5720/22.structural_variation
module load miniconda3/24.1.2-py310
conda activate sv

# Map Mo17 long reads to the B73 reference sequence
minimap2 -ax map-hifi -t 48 -R '@RG\tID:Mo17\tSM:Mo17' B73_chr2_5M.fa Mo17_chr2_5M.fastq.gz | samtools sort - | samtools view -bq 40 > B73_chr2_5M.Mo17.bam

# Map B73 long reads to the B73 reference sequence
minimap2 -ax map-hifi -t 48 -R '@RG\tID:B73\tSM:B73' B73_chr2_5M.fa B73_chr2_5M.fastq.gz | samtools sort - | samtools view -bq 40 > B73_chr2_5M.B73.bam
```

Use Sniffles v2 to identify SVs that are >= 10 bp with at least 3 supporting reads. Filtering out unreliable read alignments with mapping quality below 40 helps to control the calling confidence.
Here we perform multi-sample SV calling.
```
# index both bam files with one command
samtools index -M B73_chr2_5M.Mo17.bam B73_chr2_5M.B73.bam

# Create .snf for each sample
sniffles --input B73_chr2_5M.B73.bam --snf B73_chr2_5M.B73.snf
sniffles --input B73_chr2_5M.Mo17.bam --snf B73_chr2_5M.Mo17.snf

# Joint SV calling
sniffles --input B73_chr2_5M.Mo17.snf B73_chr2_5M.B73.snf \
	--vcf B73_chr2_5M.PacBio.SV.raw.vcf \
	--reference B73_chr2_5M.fa \
	--minsvlen 10 --minsupport 3 --mapq 40
```

Next we can inspect the file with e.g.:
```
less -S B73_chr2_5M.PacBio.SV.raw.vcf
grep -vc '#' B73_chr2_5M.PacBio.SV.raw.vcf
```

How many raw SVs did you detect?

#### SV filtering ####
Let's perform some filtering to control false positives.
First, keep SVs with a quality score of 50 or higher.
Then, remove SVs longer than the read length to reduce errors introduced by alignment uncertainty. Here we use 20 kb.
Last, keep only homozygous SVs, since these are maize inbred lines with very low heterozygosity.
```
bcftools filter -i 'QUAL>=50 && ABS(INFO/SVLEN)<=20000 && GT="1/1"' B73_chr2_5M.PacBio.SV.raw.vcf > B73_chr2_5M.PacBio.SV.vcf
grep -vc '#' B73_chr2_5M.PacBio.SV.vcf
```

How many SVs did you retain?

### Structural variant comparison ###
Now that we generated Assembly, Illumina, and PacBio based SV calls, it is time to compare them. One tool that you can use for this very easily is SURVIVOR. SURVIVOR is a very simple method to compare SVs, but it also includes multiple other methods that might be useful.

For SURVIVOR, we want to use the merge option. The merge option requires a file listing all the VCF files that you want to compare. Thus, we generate the file like this:
```
cd /fs/scratch/PAS3124/name_data/MG5720/22.structural_variation
module load miniconda3/24.1.2-py310
conda activate sv
ls B73_chr2_5M.Illumina.InDel.vcf > vcf_files
ls B73_chr2_5M.WGA.SV.vcf >> vcf_files
ls B73_chr2_5M.PacBio.SV.vcf >> vcf_files
```

Next, we can initiate the comparison with a 100 bp wobble, requiring that we only merge SVs that agree on the SV type. Furthermore, we will only take variants of 1 bp or longer into account.
```
SURVIVOR merge vcf_files 100 1 1 0 0 0 B73_chr2_5M.merged.SV.vcf
```

Let's check how good/bad the overlap is:
```
cat vcf_files
perl -ne 'print "$1\n" if /SUPP_VEC=([^,;]+)/' B73_chr2_5M.merged.SV.vcf | sort | uniq -c
```

As you can see, you will get the pattern and the number of times the pattern occurs. The first number is the number of times it can be observed in the VCF file. The second number is the pattern (0 or 1 depending on whether it was observed in each file, in the order of `vcf_files`).
