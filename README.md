# Introduction
GenEra is an easy-to-use, low-dependency and highly customizable command-line tool that estimates the age of the earliest common ancestor of protein-coding genes though genomic phylostratigraphy (Domazet-Lošo et al., 2007). GenEra takes advantage of DIAMOND’s speed and sensitivity to search for homolog genes throughout the entire NR database, and combines these results with the NCBI taxonomy to assign an origination date for each gene and gene family in a query species. GenEra can also incorporate protein data from external sources to enrich the analysis, it can search for proteins within nucleotide data (i.e., genome/transcriptome assemblies) using MMseqs2 to improve the classification of orphan genes, it automatically collapses phylostrata that lack genomic data for its inclusion in the analysis, and it calculates a taxonomic representativeness score to assess the reliability of assigning a gene to a specific phylostratum. Additionally, it can calculate the probability of a gene origination age to appear younger than it really is due to homology detection failure.

# Dependencies

GenEra requires the following software dependencies:

-	DIAMOND (https://github.com/bbuchfink/diamond)
-	NCBItax2lin (https://github.com/zyxue/ncbitax2lin)
-	MCL (https://github.com/micans/mcl)
-	MMseqs2 (https://github.com/soedinglab/MMseqs2) (optional for protein-against-nucleotide sequence search)
-	abSENSE (https://github.com/caraweisman/abSENSE) (optional to calculate homology detection failure probabilities)
-	NumPy (https://numpy.org/) and SciPy (https://scipy.org/) (needed to run abSENSE in step 4)

Additionally, GenEra requires a locally installed NR database for DIAMOND, as well as internet connection and access to the taxonomy dump from the NCBI.

# Installation

For an easy conda installation, copy and paste this in your terminal:

```console
git clone https://github.com/josuebarrera/GenEra.git
cd GenEra
conda create -n genEra python=3.7
conda activate genEra
conda install -c bioconda diamond
conda install -c bioconda mcl
pip install -U ncbitax2lin
conda install -c conda-forge -c bioconda mmseqs2
conda install -c anaconda scipy
CONDABIN=$(which ncbitax2lin | sed 's/ncbitax2lin//g') && mv genEra ${CONDABIN} && mv Erassignation.sh ${CONDABIN}
```

Otherwise, you can install the dependencies independently and then include both genEra and Erassignation.sh to your PATH.

# Setting up the databases

First, download the nr database (warning: this is a huge FASTA file):
```console
wget ftp://ftp.ncbi.nlm.nih.gov/blast/db/FASTA/nr{.gz,.gz.md5} && md5sum -c *.md5
gunzip nr.gz
```
NOTE: Alternatively, you can download any other database whose sequence IDs can be traced back to the NCBI Taxonomy.

Then download “prot.accession2taxid” from the NCBI webpage:
```console
wget ftp://ftp.ncbi.nih.gov:21/pub/taxonomy/accession2taxid/prot.accession2taxid.gz && gunzip prot.accession2taxid.gz
```
Then download the taxonomy dump from the NCBI:
```console
wget -N ftp://ftp.ncbi.nlm.nih.gov/pub/taxonomy/taxdump.tar.gz
mkdir -p taxdump && tar zxf taxdump.tar.gz -C ./taxdump
```
Finally, create a local nr database (another huge file):
```console
diamond makedb \
 --in nr \
 --db nr \
 --taxonmap prot.accession2taxid \
 --taxonnodes taxdump/nodes.dmp \
 --taxonnames taxdump/names.dmp \
 --memory-limit 100
```
You can eliminate “prot.accession2taxid”, but keep the taxdump, as GenEra will use it later on.

# Quick start for the impatient

The basic usage of GenEra is:
```console
genEra -q [query_sequences.fasta] -t [query_taxid] -b [path/to/nr] -d [path/to/taxdump]
```
However, several optional arguments can be included to get the best results possible:
```console
genEra -q [query_sequences.fasta] -t [query_taxid] -b [path/to/nr] -d [path/to/taxdump] \ 
-a [protein_list.tsv] -f [nucleotide_list.tsv] -s [evolutionary_distances.tsv] -i [true] -n [many threads]
  ```

# Arguments and input files

### GenEra always requires these two inputs:

  -q: A standard FASTA file containing all the protein sequences of your species of interest. Make sure all your sequence headers are unique.

  -t: The NCBI taxonomy ID of your species of interest (it can be easily found at https://www.ncbi.nlm.nih.gov/taxonomy).

### Additionally, GenEra requires one of the following two files:

  -b: A locally installed NR database for DIAMOND (or any other database whose sequences can be automatically traced back to NCBI taxids).

  -p: A pre-generated BLAST/DIAMOND/MMseqs2 table. This file is automatically generated by GenEra in the first step of the pipeline, and can be given to GenEra in case the user decides to re-run the pipeline (e.g., fine-tunning of the parameters). The first column contains the query sequence IDs (qseqid), the second coulmn contains the tarjet sequence IDs (sseqid), the third column contains the e-value of the matching alignment (evalue), the fourth column contains the bitscore of the alignment (bitscore) and the fifth column contains the taxonomy ID of the tarjet sequence (staxids). 

### And one of the following three files:

  -d: The location of the NCBI taxonomy files, colloquially known as the “taxdump”. Make sure the taxdump directory contains the files “nodes.dmp” and “names.dmp”. This files are used by NCBItax2lin to create a “ncbi_lineages” file.

  -r: The location of the uncompressed “ncbi_lineages” file generated by NCBItax2lin. Once the user runs GenEra for the first time (or runs NCBItax2lin independently), this file can be used to save some time during step 2 of the pipeline. This file is a comma-delimited table with each row representing a specific lineage, and each column representing the taxonomic hierarchies to which that lineage belongs. This file will be automatically modified by genEra to rearrange the columns in hierarchical order.

  -c: Custom "ncbi_lineages" file that is already tailored for the query species. GenEra modifies the raw “ncbi_lineages” file so that the taxid of the query species appears in the first column, and the taxonomic hierarchies of the query species are rearranged from the species level all the way back to the last universal common ancestor (termed “cellular organisms” by the NCBI taxonomy). The file is also modified by GenEra to collapse the phylostrata (i.e., the taxonomic levels) that lack the necessary genomic data to be useful in the analysis. Once this file is generated, the user can re-use this file if they want to run GenEra on the same species with different parameters. By default, GenEra will search for the correct hierarchical order of the query organism’s phylostrata directly from the NCBI webpage. If the user machine is unable to access the NCBI webpage, GenEra will attempt to infer the correct order of the phylostrata directly from the “ncbi_lineages” file. However, there are some taxonomic hierarchies that the NCBI labels as “clades” (e.g., Embryophyta in the plant lineage), which GenEra cannot automatically assign to their correct taxonomic level when working offline. This option is also useful if the user wants to modify the taxonomic hierarchies that were originally assigned by the NCBI (e.g., an outdated taxonomy that does not reflect the phylogenetic relationships of the query species). This custom table can be easily generated by printing the desired columns of the “ncbi_lineages” file with awk:
```console
awk -F'"' -v OFS='' '{ for (i=2; i<=NF; i+=2) gsub(",", "", $i) } 1' ncbi_lineages_[date].csv | awk -F "," '{ print $1","$8","$7","$6"," ... }' > custom_table.csv
```
### The user can also incorporate four other inputs, which are optional but can potentially improve the final age assignation:

-a: Tab-delimited table with additional proteins to be included in the analysis. This option is particularly useful if the user wants to include proteins from species that are absent from the nr database, such as newly annotated genomes or transcriptomes. It can also help fill the gaps of phylostrata that would otherwise be collapsed due to lack of available genomes in the database. Importantly, each protein in this custom dataset should have unique identifiers, otherwise GenEra will not work properly during the taxonomic assignation of the Diamond hits. The table format consists of the location of one protein FASTA file for each additional species in the first column and the NCBI taxonomy ID of that species in the second column:

		   /path/to/species_1.fasta	taxid_1
		   /path/to/species_2.fasta	taxid_2
		   /path/to/species_3.fasta	taxid_3

-f: Table with additional nucleotide sequences (e.g., non-annotated genome assemblies) to be searched against your query proteins with MMseqs2 in a "tblastn" fashion. This option is extremely useful to improve the detection of orphan genes, since errors in the genome annotations can always lead to the spurious detection of taxonomically-restricted genes (Basile et al., 2019; Weisman et al., 2022). Importantly, each contig/scaffold/chromosome in this dataset should have a unique identifier (e.g., avoid having multiple “>chr1” sequences throughout your genome assemblies). The table format consists of the location of one nucleotide FASTA file for each genome/transcriptome assembly in the first column and the NCBI taxonomy ID of that species in the second column:

		   /path/to/assembly_1.fasta	taxid_1
		   /path/to/assembly_2.fasta	taxid_2
		   /path/to/assembly_3.fasta	taxid_3

-s: Table with pairwise evolutionary distances, calculated as substitutions/site, between several species in a phylogeny and the query species. 
