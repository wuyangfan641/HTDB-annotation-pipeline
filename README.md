# HTDB-annotation-pipeline
> ***Title: A near-comprehensive multi-omics HardTick database anchored to reference genomes from NCBI***
>  
> ***Author: By Yangfan Wu***
> 
> ***Institution: Wanna Medical College***
>
> ***Official accounts: https://mp.weixin.qq.com/s/Z3QQj62cOeFPlakYFNO1Eg***
>
> ***Contact us: http://www.waspomics.com/team/***
------

### 01 Prerequisites
#### 1.1 Software
+ BUSCO (https://busco.ezlab.org/) (compleasm(https://github.com/huangnengCSU/compleasm), compleasm: a faster and more accurate reimplementation of BUSCO)
+ CheckM2 (https://github.com/chklovski/checkm2)
+ EDTA (https://github.com/oushujun/EDTA)
+ HiTE (https://github.com/CSU-KangHu/HiTE)
+ egapx (https://github.com/ncbi/egapx)
+ Bakta (https://github.com/oschwengers/bakta)
+ geNomad (https://github.com/apcamargo/genomad)
+ LoVis4u (https://github.com/art-egorov/lovis4u)


#### 1.2 Database
+ arachnida_odb10 (https://busco-data.ezlab.org/v5/data/lineages/arachnida_odb10.2024-01-08.tar.gz)
+ db-light (https://zenodo.org/record/7669534/files/db-light.tar.gz?download=1)
+ genomad_db (https://zenodo.org/records/14886553/files/genomad_db_v1.9.tar.gz?download=1)
+ checkm2_database (https://zenodo.org/api/files/fd3bc532-cd84-4907-b078-2e05a1e46803/checkm2_database.tar.gz)

#### 1.3 Reference genome
+ genome (https://www.ncbi.nlm.nih.gov/datasets/genome/)
+ transcript (https://www.ncbi.nlm.nih.gov/sra/)
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
perl EDTA.pl --genome ../tickdb/genome.fna --step all --overwrite 1 -t 12 --sensitive 1 --anno 1 --force 1 --evaluate 1
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
mamba create -n egapx -c bioconda python pyyaml nextflow singularity
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
  annotation_provider: GenBank submitter
  locus_tag_prefix: egapxtmp
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
### 06 LoVis4u: a locus visualization tool for comparative genomics and coverage proffles
- genome.gff
- The development version is available at github :

```
git clone https://github.com/art-egorov/lovis4u.git
cd lovis4u
python3 -m pip install --upgrade pip
python3 -m pip install setuptools wheel
python3 setup.py sdist
python3 -m pip install -e . -i https://pypi.tuna.tsinghua.edu.cn/simple
lovis4u --linux
```

some details about loVis4u:
```
[MANDATORY ARGUMENTS]
-gff <folder>
    Path to a folder containing extended gff files.
    Each gff file should contain corresponding nucleotide sequence.
    (designed to handle pharokka produced annotation files).
 OR
-gb <folder>
    Path to a folder containing genbank files.
-------------------------------
[OPTIONAL ARGUMENTS | DATA PROCESSING]
-w, --windows <locus_id1:start1:end1:strand [locus_id1:start1:end1:strand ...]>
    Specify window of visualisation (coordinates) for a locus or multiple loci
-bg, --bedgraphs <bedGraph_file1 [bedgGaph_file2 ...]>
    Space separated list of paths to bedGraph files to plot coverage profiles.
    (>=1 file) ! Can be applied only for single locus.
-bw, --bigwigs <bigWig_file1 [bigWig_file2 ...]>
    Space separated list of paths to bigWig files to plot coverage profiles.
    (>=1 file) ! Can be applied only for single locus.
-bgl, --bedgraph-labels <bedgraph_label1 [bedgraph_label2 ...]>
    List of labels for bedgraph/bigWig tracks (order the same as order of bedgraph/bigWig files)
    By default basename of files will be used.
-bgc, --bedgraph-colours <bedgraph_colour1 [bedgraph_colour2 ...]>
    List of colours for bedgraph/bigWig tracks (order the same as order of bedGraph/bigWig files)
    Each value can be either HEX code of colour name (e.g. pink, blue, etc (from the palette file))
-gc, --gc-track
    Show GC content track. ! Can be applied only for single locus.
-gc_skew, --gc_skew-track
    Show GC skew track. ! Can be applied only for single locus.
-ufid, --use-filename-as-id
    Use filename (wo extension) as track (contig) id instead
    of the contig id written in the gff/gb file.
-alip, --add-locus-id-prefix
    Add locus id prefix to each feature id.
-laf, --locus-annotation-file <file path>
    Path to the locus annotation table.
    (See documentation for details)
-faf, --feature-annotation-file <file path>
    Path to the feature annotation table.
    (See documentation for details)
-mmseqs-off, --mmseqs-off
    Deactivate mmseqs clustering of proteomes of loci.
-hmmscan, --run-hmmscan
    Run hmmscan search for additional functional annotation.
-dm, --defence-models <DefenseFinder|PADLOC|both>
    Choose which defence system database to use for hmmscan search
    [default: both (DefenseFinder and PADLOC)]
-hmm, --add-hmm-models <folder_path [name]>
    Add your own hmm models database for hmmscan search. Folder should
    contain files in HMMER format (one file per model). Usage: -hmm path [name].
    Specifying name is optional, by default it will be taken from them folder name.
    If you want to add multiple hmm databases you can use this argument several
    times: -hmm path1 -hmm path2.
-omh, --only-mine-hmms
    Force to use only models defined by user with -hmm, --add-hmm-models parameter.
-kdn, --keep-default-name
    Keep default names and labels for proteins that have hits with
    hmmscan search. [default: name is replaced with target hmm model name]
-kdc, --keep-default-category
    Keep default category for proteins that have hits with hmmscan
    search. [default: category is replaced with database name]
-salq, --show-all-labels-for-query
    Force to show all labels for proteins that have hits to any database with hmmscan search.
    [default: False]
-cl-owp, --cluster-only-window-proteins
    Cluster only proteins that are overlapped with the visualisation windows, not all.
-fv-off, --find-variable-off
    Deactivate annotation of variable or conserved protein clusters.
-cl-off, --clust_loci-off
    Deactivate defining locus order and using similarity based hierarchical
    clustering of proteomes.
-oc, --one-cluster
    Consider all sequences to be members of one cluster but use clustering
    dendrogram to define the optimal order.
-reorient_loci, --reorient_loci
    Auto re-orient loci (set new strands) if they are not matched.
    (Function tries to maximise co-orientation of homologous features.)
-------------------------------
[OPTIONAL ARGUMENTS | LOCUS VISUALISATION]
-sgc-off, --set-group-colour-off
    Deactivate auto-setting of feature fill and stroke colours.
    (Pre-set colours specified in feature annotation table will be kept.)
-sgcf, --set-group-colour-for <feature_group1 [feature group2 ...]>
    Space-separated list of feature groups for which colours should be set.
    [default: variable, labeled]
-scc, --set-category-colour
    Set category colour for features and plot category colour legend.
-cct, --category-colour-table <file path>
    Path to the table with colour code for categories.
    Default table can be found in lovis4u_data folder.
-lls, --locus-label-style <id|description|full>
    Locus label style based on input sequence annotation.
-llp, --locus-label-position <left|bottom|top_left|top_center>
    Locus label position on figure.
-safl, --show-all-feature-labels
    Display all feature labels.
-sflf, --show-feature-label-for  <feature_group1 [feature group2 ...]>
    Space-separated list of feature groups for which label should be shown.
    [default: variable, labeled]
-sfflf, --show-first-feature-label-for <feature_group1 [feature group2 ...]>
    Space-separated list of feature group types for which label will be displayed
    only for the first occurrence of feature homologues group.
    [default: shell/core]
-snl, --show-noncoding-labels
    Show all labels for non-coding features. [default: False]
-sfnl, --show-first-noncoding-label
    Show labels only for the first occurrence for non-coding features.
    [default: False]
-ifl, --ignored-feature-labels <feature_label1 [feature_label2 ...]>
    Space-separated list of feature names for which label won't be shown.
    [default: hypothetical protein, unknown protein]
-sxa, --show-x-axis
    Plot individual x-axis for each locus track.
-hix, --hide-x-axis
    Do not plot individual x-axis for each locus track.
-dml, --draw-middle-line
    Draw middle line for each locus.
-mm-per-nt, --mm-per-nt <float value>
    Scale which defines given space for each nt cell on canvas.
    [default: 0.05]
-fw, --figure-width <float value>
    Output figure width in mm.
-------------------------------
[OPTIONAL ARGUMENTS | ADDITIONAL TRACKS]
-hl, --homology-links
    Draw homology link track.
-slt, --scale-line-track
    Draw scale line track.
-------------------------------
[OPTIONAL ARGUMENTS | OTHERS]
-o <name>
    Output dir name. It will be created if it does not exist.
        [default: lovis4u_{current_date}; e.g. uorf4u_2022_07_25-20_41]
--pdf-name <name>
    Name of the output pdf file (will be saved in the output folder).
    [default: lovis4u.pdf]
-c <standard|<file.cfg>
    Path to a configuration file or name of a pre-made config file
    [default: standard]
```
The LoVis4u tool provides a fast and automated genomic visualization solution by combining a command-line interface and Python API. 

------
### 07 Automated annotation of Tick-Borne Virus Genomes by geNomad
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
