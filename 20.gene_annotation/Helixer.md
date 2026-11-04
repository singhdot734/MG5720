## Using Deep Learning for ab initio gene predictions ##

#### Helixer GitHub ####
This is the original Helixer that includes model training and prediction  
https://github.com/usadellab/Helixer  
https://github.com/gglyptodon/helixer-docker  

#### HelixerLite GitHub ####
This is a lightweight and easy-to-install Helixer that only includes model prediction  
https://github.com/nextgenusfs/helixerlite  

### Set up ###
#### 1. Request compute resources ####
```
sinteractive -A PAS3124 -c 48 -t 1:00:00
```

#### 2. Set up your course folder ####
Do this once. Replace `name.#` with your OSU name.# in all commands below.
```
cd /fs/ess/PAS3124/
mkdir name.#
cd name.#
git clone https://github.com/singhdot734/MG5720.git
```

If you already have the course folder, update it instead.
```
cd /fs/ess/PAS3124/name.#/MG5720
# sync to the latest course files; your data and outputs are kept
git fetch origin
git reset origin/main
git restore .
```

#### 3. Link the genome and its reference annotation ####
```
cd /fs/ess/PAS3124/name.#/MG5720/20.gene_annotation
ln -s /fs/ess/PAS3124/class/data/B73_chr2_5M.fa
ln -s /fs/ess/PAS3124/class/data/B73_chr2_5M.Zm00001eb.1.genes.gff3
less B73_chr2_5M.fa
```

#### 4. Create conda environments ####
```
module load miniconda3/24.1.2-py310

# "mapping" environment: lectures 21 and 25 install their tools into it, so keep this name
# (created only if it doesn't exist yet, so re-running this block never wipes those tools)
conda env list | grep -q "^mapping " || conda create -n mapping -y

# "helixer" environment for this lecture
conda env create -f helixer.yml
conda activate helixer

# install HelixerLite, use ONE of the two options
# option 1 (recommended): pinned versions tested for this class
pip install -r helixer_pip.yml
# option 2: latest version (keep gfftk at 25.6.10; newer gfftk writes wrong UTRs)
python -m pip install helixerlite gfftk==25.6.10

# test if you have helixerlite installed
helixerlite -h
```

### Annotate Maize sequences with different models ###
Run the four models in the background. Together they take about 5 minutes and up to ~95 GB of memory, which the 48-CPU session above provides (OSC gives memory per CPU). HelixerLite accepts `-c 12` but ignores it; TensorFlow picks its own thread count.
```
cd /fs/ess/PAS3124/name.#/MG5720/20.gene_annotation
module load miniconda3/24.1.2-py310
conda activate helixer
nohup helixerlite --fasta B73_chr2_5M.fa --lineage land_plant --out B73_chr2_5M.land_plant.output.gff3 -c 12 > land_plant.log 2>&1 &
nohup helixerlite --fasta B73_chr2_5M.fa --lineage fungi --out B73_chr2_5M.fungi.output.gff3 -c 12 > fungi.log 2>&1 &
nohup helixerlite --fasta B73_chr2_5M.fa --lineage vertebrate --out B73_chr2_5M.vertebrate.output.gff3 -c 12 > vertebrate.log 2>&1 &
nohup helixerlite --fasta B73_chr2_5M.fa --lineage invertebrate --out B73_chr2_5M.invertebrate.output.gff3 -c 12 > invertebrate.log 2>&1 &
```

Check progress. The runs are done when `jobs` shows nothing, all four gff3 files exist, and each log ends with `module finished`. TensorFlow messages about CUDA, cuDNN or TensorRT (such as `CUDA_ERROR_NO_DEVICE`) are normal on CPU nodes; ignore them.
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
