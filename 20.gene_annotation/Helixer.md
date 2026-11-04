## Using Deep Learning for ab initio gene predictions

#### Helixer GitHub ###
This is the original Helixer that includes model training and prediction  
https://github.com/usadellab/Helixer  
https://github.com/gglyptodon/helixer-docker  

#### HelixerLite GitHub ###
This is a lightweight and easy-to-install Helixer that only includes model prediction  
https://github.com/nextgenusfs/helixerlite  

### Set up the tool ###
#### 1. Request compute resources ####
```
sinteractive -A PAS3124 -n 48 -t 1:00:00
```

#### 2. Update your course folder
```
cd ~/MG5720/name.#/MG5720  
git fetch origin  
git reset origin/main  # sync to the latest course files; your data and outputs are kept  
cd 20.gene_annotation  
ls -l B73_chr2_5M.fa  # the 5 Mb maize B73 sequence to annotate
```

#### 3. Install HelixerLite ###
```
# set up the base environment
module load miniconda3/24.1.2-py310
conda env create -f helixer.yml
conda activate helixer

# install HelixerLite, use ONE of the two options
pip install -r helixer_pip.yml  # option 1: pinned versions tested for this class (recommended)
python -m pip install helixerlite  # option 2: latest version

# test if you have helixerlite installed
helixerlite -h
```

### Annotate Maize sequences with different models ###
Run the four models in the background. Each uses 12 CPUs, so all four fit in the 48 CPUs requested above.
```
module load miniconda3/24.1.2-py310
conda activate helixer
nohup helixerlite --fasta B73_chr2_5M.fa --lineage land_plant --out B73_chr2_5M.land_plant.output.gff3 -c 12 > land_plant.log 2>&1 &  
nohup helixerlite --fasta B73_chr2_5M.fa --lineage fungi --out B73_chr2_5M.fungi.output.gff3 -c 12 > fungi.log 2>&1 &  
nohup helixerlite --fasta B73_chr2_5M.fa --lineage vertebrate --out B73_chr2_5M.vertebrate.output.gff3 -c 12 > vertebrate.log 2>&1 &  
nohup helixerlite --fasta B73_chr2_5M.fa --lineage invertebrate --out B73_chr2_5M.invertebrate.output.gff3 -c 12 > invertebrate.log 2>&1 &
```

Check progress. The runs are done when `jobs` shows nothing and all four gff3 files exist.
```
jobs  
tail -n 3 *.log  
ls -lh B73_chr2_5M.*.output.gff3
```

### In-class exercise: count gene models and exons ###
Column 3 of a GFF3 file is the feature type. Count the `gene` and `exon` lines of each output and fill in the table.
```
printf "Model\tGene_models\tExons\tAvg_exons_per_gene\n"  
for model in land_plant fungi vertebrate invertebrate; do  
  awk -F'\t' -v model=$model '$3=="gene"{g++} $3=="exon"{e++} END{printf "%s\t%d\t%d\t%.2f\n", model, g, e, (g ? e/g : 0)}' B73_chr2_5M.$model.output.gff3  
done
```

Compare with the reference annotation (Zm00001eb.1) of the same region. The reference has alternative transcripts (several mRNAs per gene), so its exon count includes all isoforms.
```
awk -F'\t' '$3=="gene"{g++} $3=="mRNA"{t++} $3=="exon"{e++} END{printf "Reference\tgenes=%d\tmRNAs=%d\texons=%d\tavg_exons_per_gene=%.2f\tavg_exons_per_mRNA=%.2f\n", g, t, e, e/g, e/t}' B73_chr2_5M.Zm00001eb.1.genes.gff3
```

### Download and Install IGV on your own computer ###
https://igv.org/download/html/oldtempfixForDownload.html
