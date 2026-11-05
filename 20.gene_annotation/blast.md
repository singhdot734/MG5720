## Set up your environment ##
First request compute resources and update your course folder (steps 1 and 2 in [Helixer.md](Helixer.md)). The BLAST practice uses the same `20.gene_annotation` folder.

#### Create and activate a conda environment called "mapping"
You can name it what ever you like, just something simple and easy to remember
```
module load miniconda3/24.1.2-py310  
conda create -n mapping -y  
conda activate mapping  
```

#### Install blast to the mapping environment
```
conda install -c bioconda blast  
blastn -h
blastn -help
```

#### Create a softlink of the genome
Enter your course folder; replace name_data with your folder name
```
cd /fs/scratch/PAS3124/name_data/MG5720/20.gene_annotation
ln -s /fs/ess/PAS3124/class/data/Rice_AGIS1.fasta  
less Rice_AGIS1.fasta  
```

#### Explore blast output formats
```
less blast_test1.fa  
blastn -query blast_test1.fa -subject Rice_AGIS1.fasta -outfmt 0  
blastn -query blast_test1.fa -subject Rice_AGIS1.fasta -outfmt 6  
blastn -query blast_test1.fa -subject Rice_AGIS1.fasta -outfmt 7  
blastn -query blast_test1.fa -subject Rice_AGIS1.fasta -outfmt '6 qseqid sseqid pident length qlen qcovs evalue'
```

#### Run blast with many sequences
```
less B73_chr2_5M.Zm00001eb.1.genes.CDS.fa  
blastn -query B73_chr2_5M.Zm00001eb.1.genes.CDS.fa -subject Rice_MSU7.fasta -outfmt '6 qseqid sseqid pident length qlen qcovs evalue' > B73_chr2_5M.Zm00001eb.1.genes.CDS.fa.out  
```

#### Filter blast results
```
awk '{if ($6 > 80 && $3 > 70) print $0}' B73_chr2_5M.Zm00001eb.1.genes.CDS.fa.out | less  
```


