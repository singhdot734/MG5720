# Class 9
## Creating list of shortest and longest human genes
1. Starting with the Ensembl GTF, create a new GTF that has two additional columns with gene_biotype and gene_id using this code: `awk -F'\t' '$3=="gene" {match($9, /gene_biotype "([^"]+)"/, biotype); match ($9, /gene_id "([^"]+)"/, gene_id); p
rint $0 "\t" biotype[1] "\t" gene_id[1];}' GRCh38.115.gtf > GRCh38.115.biotype.gene_id.gtf`
2. Create a text file with gene lengths: `awk -F'\t' '$3=="gene" && $10=="protein_coding" {print $11 "\t" $5-$4}' GRCh38.115.biotype.gene_id.gtf > gene_id_
length_GRCh38.115.txt`
3. Sort by column 2: `cat gene_id_length_GRCh38.115.txt | sort -t $'\t' -k2,2V > sorted_gene_id_length_GRCh38.115.txt`
4. Top 10% shortest (roughly) human genes: `cat sorted_gene_id_length_GRCh38.115.txt | head -2200 | tail -2000 | cut -f 1 > pc_genes_short10_ensg.txt`
5. Top 10% longest (roughly) human genes: `cat sorted_gene_id_length_GRCh38.115.txt | tail -2200 | head -2000 | cut -f 1 > pc_genes_long10_ensg.txt`
6. Download each of these files to your computer.

## Determine which functionally related group of genes are among the shortest and longest human genes
1. [Go to Pantherdb](https://www.pantherdb.org/)
2. Perform analysis as shown in pptx slides.
3. Same analysis can also be performed at [PAN-GO](https://functionome.geneontology.org/) or [Geneontology.org](https://geneontology.org/).
 

