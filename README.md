# HTDB-annotation-pipeline

### 01 Prerequisites
#### 1.1 Software
+ BUSCO (https://busco.ezlab.org/) (compleasm(https://github.com/huangnengCSU/compleasm), compleasm: a faster and more accurate reimplementation of BUSCO)
+ CheckM2 (https://github.com/chklovski/checkm2)
+ EDTA (https://github.com/oushujun/EDTA)
+ HiTE (https://github.com/CSU-KangHu/HiTE)
+ egapx (https://github.com/ncbi/egapx)
+ Bakta (https://github.com/oschwengers/bakta)
+ geNomad (https://github.com/apcamargo/genomad)


#### 1.2 Database
+ arachnida_odb10 (https://busco-data.ezlab.org/v5/data/lineages/arachnida_odb10.2024-01-08.tar.gz)
+ db-light (https://zenodo.org/record/7669534/files/db-light.tar.gz?download=1)
+ genomad_db (https://zenodo.org/records/14886553/files/genomad_db_v1.9.tar.gz?download=1)
+ checkm2_database (https://zenodo.org/api/files/fd3bc532-cd84-4907-b078-2e05a1e46803/checkm2_database.tar.gz)

------

### 02 Genome assessment
#### 2.1 BUSCO v5.6.0
- genome.fna
- arachnida_odb10

```shell
busco -i ./tickdb/genome.fna -l ./arachnida_odb10 -o ./busco_output -m genome --cpu 12 --offline
```
```bash
cat ./busco_output/short_summary.specific.arachnida_odb10.out.txt
```
or you can try compleasm, a faster and more accurate reimplementation of BUSCO
#### 2.2 compleasm
```bash
python compleasm.py run -t 16 -l arachnida -L ./arachnida_odb10 -a genome.fna -o compleasm_output
```
Note: the organization of the lineage file downloaded by compleasm is different from that of BUSCO.

#### 2.3 CheckM2
- bacteriadb/
- checkm2_database

The main use of CheckM2 is to predict the completeness and contamination of metagenome-assembled genomes (MAGs) and single-amplified genomes (SAGs), although it can also be applied to isolate genomes.

You can give it a folder with FASTA files using --input and direct its output with --output-directory:
```bash
checkm2 predict --threads 16 --input ./bacteriadb --database_path ./CheckM2_database/uniref100.KO.1.dmnd --output_directory ./checkm2_output
```
By default, the output folder will have a tab-delimited file `quality_report.tsv` containing the completeness and contamination information for each bin. You can also print the results to stdout by passing the `--stdout` option to `checkm predict`.

------

### 03 Repeat annotation and genome mask
#### 3.1 EDTA 
- genome.fna

You got a genome and you want to get a high-quality TE annotation:
```bash
perl EDTA.pl --genome ../tickdb/genome.fna --step all --overwrite 1 -t 12 --sensitive 1 --anno 1 --force 1 evaluate 1
```
Convert hard masking into soft masking:
```bash
perl ./util/make_masked.pl -genome genome.fna -minlen 80 -hardmask 0 -t 10 -rmout genome.fna.mod.EDTA.TEanno.out
```
#### 3.2 HiTE
- genome.fna

Use the TE library generated by HiTE to annotate the genome. This will produce annotation files such as `HiTE.out`, `HiTE.gff`, and `HiTE.tbl`. 
```bash
python main.py --genome ../tickdb/genome.fna --outdir ../hite_output --thread 12 --annotate 1 --plant 0
```

HiTE outputs many temporary files, which allow you to quickly restore the previous running state (use `--recover 1`) in case of any interruption during the running process. If
the pipeline completes successfully, the output directory should look like the following:
```shell
output_dir/
├── longest_repeats_*.fa
├── longest_repeats_*.flanked.fa
├── confident_tir_*.fa
├── confident_helitron_*.fa
├── confident_non_ltr_*.fa
├── confident_other_*.fa
├── confident_ltr_cut.fa.cons
├── confident_TE.cons.fa
├── HiTE.out (require `--annotate 1`)
├── HiTE.gff (require `--annotate 1`)
└── HiTE.tbl (require `--annotate 1`)
```
------

### 04 Structural annotation of the tick genome by EGAPx
- genome.rename.fa.masked
- input_tick.yaml
- egapx_0.3.2-alpha.sif
- local_cache

Create a environment called egapx:
```bash
mamba create -n egapx -c bioconda python pyyaml nextflow
mamba activate egapx
```

Download the required mirror:
```bash
singularity pull docker://docker.1ms.run/ncbi/egapx:0.3.2-alpha
```

Clone the EGAPx repo:
```bash
git clone https://github.com/ncbi/egapx.git
```

Download the required database:
```bash
cd egapx
python ui/egapx.py -dl -lc ../local_cache
```
or download it mannually:
```shell
mkdir local_cache
cd local_cache
mkdir gnomon ortholog_references target_proteins taxonomy reference_sets misc
rsync --copy-links --recursive --times --verbose rsync://ftp.ncbi.nlm.nih.gov/genomes/TOOLS/EGAP/support_data/gnomon/2 gnomon/
rsync --copy-links --recursive --times --verbose rsync://ftp.ncbi.nlm.nih.gov/genomes/TOOLS/EGAP/support_data/ortholog_references/2 ortholog_references/
rsync --copy-links --recursive --times --verbose rsync://ftp.ncbi.nlm.nih.gov/genomes/TOOLS/EGAP/support_data/target_proteins/2 target_proteins/
rsync --copy-links --recursive --times --verbose rsync://ftp.ncbi.nlm.nih.gov/genomes/TOOLS/EGAP/support_data/taxonomy/1 taxonomy/
rsync --copy-links --recursive --times --verbose rsync://ftp.ncbi.nlm.nih.gov/genomes/TOOLS/EGAP/support_data/reference_sets/2 reference_sets/
rsync --copy-links --recursive --times --verbose rsync://ftp.ncbi.nlm.nih.gov/genomes/TOOLS/EGAP/support_data/misc/2 misc/
```

Input to EGAPx is in the form of a YAML file. The following are the _required_ key-value pairs for the input file:
  ```shell
  genome: path to assembled genome in FASTA format
  taxid: NCBI Taxonomy identifier of the target organism 
  reads: RNA-seq data
  ```
  ```
genome: https://ftp.ncbi.nlm.nih.gov/genomes/all/GCA/020/809/275/GCA_020809275.1_ASM2080927v1/GCA_020809275.1_ASM2080927v1_genomic.fna
taxid: 6954
reads:
  - https://ftp.ncbi.nlm.nih.gov/genomes/TOOLS/EGAP/sample_data/Dermatophagoides_farinae_small/SRR8506572.1
  - https://ftp.ncbi.nlm.nih.gov/genomes/TOOLS/EGAP/sample_data/Dermatophagoides_farinae_small/SRR8506572.2
  - https://ftp.ncbi.nlm.nih.gov/genomes/TOOLS/EGAP/sample_data/Dermatophagoides_farinae_small/SRR9005248.1
  - https://ftp.ncbi.nlm.nih.gov/genomes/TOOLS/EGAP/sample_data/Dermatophagoides_farinae_small/SRR9005248.2
annotation_provider: GenBank submitter
annotation_name_prefix: GCA_020809275.1
locus_tag_prefix: egapxtmp
  ```
Run EGAPx:
  ```bash
python3 ./egapx/ui/egapx.py input_tick.yaml -e singularity -w anno -o egapx_output -lc ./egapx/local_cache
  ```

  ```bash
echo "process.container = '/path_to_/egapx_0.3.2-alpha.sif'" >> egapx_config/biowulf_cluster.config
  ```

  ```bash
echo "process.container = '/path_to_/egapx_0.3.2-alpha.sif'" >> egapx_config/singularity.config
  ```

  ```bash
python3 ./egapx/ui/egapx.py input_tick.yaml -e singularity -w anno -o egapx_output -lc ./egapx/local_cache
  ```

Output:
Look at the output in the out diectory (`example_out`) that was supplied in the command line. The annotation file is called `complete.genomic.gff`. 
```shell
annot_builder_output
annotated_genome.asn
annotation_data.cmt
complete.cds.fna
complete.genomic.fna
complete.genomic.gff
complete.genomic.gtf
complete.proteins.faa
complete.transcripts.fna
nextflow.log
resume.sh
run.report.html
run.timeline.html
run.trace.txt
run_params.yaml
stats
validated
```
Description of the outputs:
* `complete.genomic.gff`: final annotation set in GFF3 format.
* `complete.genomic.gtf`: final annotation set in GTF format.
* `complete.genomic.fna`: full genome sequences set in FASTA format.
* `complete.genomic.gtf`: final annotation set in gtf format.
* `complete.cds.fna`: annotated Coding DNA Sequences (CDS) in FASTA format.
* `complete.transcripts.fna`: annotated transcripts in FASTA format (includes UTRs).
* `complete.proteins.faa`: annotated protein products in FASTA format.
* `annotated_genome.asn`: final annotation set in ASN1 format.
------
### 05 Standardized annotation of Tick-Borne Bacterial Genomes by Bakta
- genome.fna
- db-light

If required, or desired, the AMRfinderPlus-DB can also be updated manually: 
```bash
mkdir -p db-light/amrfinderplus-db
```
```bash
amrfinder_update --force_update --database db-light/amrfinderplus-db/
```

```bash
bakta --db ./db-light --output ./bakta_output ./bacteriadb/genome.fna
```
It provides dbxref-rich, sORF-including and taxon-independent annotations in machine-readable JSON & bioinformatics standard file formats for automated downstream analysis.

------

### 06 Automated annotation of Tick-Borne Virus Genomes by geNomad
- genome.fna
- genomad_db

geNomad depends on a database that contains the profiles of the markers that are used to classify sequences, their taxonomic information, their functional annotation, etc. So, you should first download the database to your current directory:
```bash
genomad download-database .
```
The database will be contained within the genomad_db directory. 

geNomad works by executing a series of modules sequentially , but we provide a convenient end-to-end command that will execute the entire pipeline for you in one go.
```bash
genomad end-to-end --cleanup --splits 8 ./virusdb/genome.fna.gz ./genomad_output ./genomad_db
```
The results will be written inside the genomad_output directory.
