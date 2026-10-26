# Functional annotation - Viromics

We will now identify viral sequences from metagenomic data. You will use the assembly from day 1 (``/data/precomputed/assembly/final.contigs.fa``). Our goals are to:

1 **Detect** viral contigs (and assign their taxonomy).
2. Estimate the **completeness** of viral genomes.
3. Functionally **annotate** a virus with different methods.


**Note:** For this practical, several tools are required for the viral identification and annotation pipeline:  
> - **geNomad**  
> - **CheckV**  
> - **Pharokka**
> - **Phold**

All these tools and the necessary databases have been preinstalled and are ready for you to use.
You can activate the environment with ``conda activate viral``.

## Step 1 - Virus discovery

First, we will run **geNomad** to identify viral sequences in our dataset. 
We will utilize the ``--enable-score-calibration`` option to compute false discovery rates (FDR). 
For this practical exercise, no specific FDR thresholds will be applied. 
Further, we will add the ``--cleanup`` flag to remove intermediate files and the ``--disable-find-proviruses`` option to avoid geNomad performing an initial prunning to remove potential contaminant host regions from proviral sequences (we will do this in the next step with CheckV). 

**Note:** The following command takes about 10 minutes. 
You can also continue with the results in ``/data/precomputed/virome/genomad_results/``.

```bash
genomad end-to-end \
    --enable-score-calibration \
    --disable-find-proviruses \
    --cleanup \
    --threads 4 \
    final.contigs.fa \
    genomad \
    /data/databases/genomad_db/
```

You can find a description of the geNomad output files [here](https://portal.nersc.gov/genomad/quickstart.html#understanding-the-outputs).

Look into ``genomad/final.contigs_summary/final.contigs_virus_summary.tsv``. 

??? done "1. How many of the contigs are identified as viral?"
    21 contigs.

??? done "2. How many single-stranded DNA viruses (ssDNA) and double-stranded DNA viruses (dsDNA) have been identified?"
    2 ssDNA (belonging to *Monodnaviria* realm), 18 dsDNA (belonging to *Duplodnaviria* realm).     
    
??? done "3. Regarding the viral taxonomy, which is the most common viral class among identified viruses?"
    *Caudoviricetes*.

**Note:** A useful resource where you can find all the information about the viral taxonomy is [here](https://ictv.global/taxonomy).  

## Step 2 - Host contamination removal and quality check

While geNomad performs well at identifying proviruses from metagenomic data, **CheckV** is specifically designed to identify host-virus boundaries with high precision. 
Further, CheckV allows to estimate the completeness of the predicted viral genomes based on their comparison to a database of complete viral genomes. 
Therefore, here we use CheckV to quality control the geNomad results and also to trim potential host regions left at the ends of proviruses. 

**Note:** This command will take about 1 minute to complete. 
You can probably run it, but you can also continue with the results in ``/data/precomputed/virome/checkv/``.

```bash
checkv \
    end_to_end \
    genomad/final.contigs_summary/final.contigs_virus.fna \
    checkv \
    -t 4 \
    -d /data/databases/checkv-db-v1.5
```

The CheckV output is described [here](https://bitbucket.org/berkeleylab/checkv/src/master/#markdown-header-output-files). Look into ``checkv/quality_summary.tsv``.

??? done "1. How many proviruses do you find and how many viruses?"
    2 proviruses, 19 viruses.

??? done "2. How many low, medium, and high quality viruses do we detect? How many complete viruses? How many of them have direct terminal repeats?"
    9 viruses with low-quality, 5 with medium-quality and 5 with high-quality.
    2 viruses are complete, both have direct terminal repeats (DTRs).

**Note:** Depending on the project's goals, it may be advisable to select only viruses predicted to be of at least medium quality. 

**Note:** CheckV provides 2 separate files with the identified proviral and viral sequences. 
To work with both of them, they could be combined into a single file: 

```
cat checkv/proviruses.fna checkv/viruses.fna > checkv/combined.fna
```

To save computing time, we will do the following analysis with only one of the High-quality viruses, contig ``k141_524``:

```
seqkit grep -p k141_524 checkv/viruses.fna >checkv/k141_524.fna
```

## Step 3 - Functional annotation

Multiple tools can be used for the functional annotation of viral genomes. 
Here we will use Pharokka and Phold for functional prediction based on sequence similarity and on structure similarity.

**Note:** This command should take less than one minute to complete, thus you can run it yourself. 
We run it in ``fast`` mode which does not run MMseqs2 and is thus faster, you can find the documentation [here](https://pharokka.readthedocs.io/en/latest/run/).

```bash
pharokka run \
    -i checkv/k141_524.fna \
    -t 4 \
    --fast \
    -o pharokka_524 \
    -d /data/databases/pharokka
```

**Note:** You can also run pharokka for a collection of virus contigs from a metagenome, using ``-m``, but this takes too long for the course:

```bash
pharokka run \
    -i checkv/combined.fna \
    -t 4 \
    -m \
    -o pharokka \
    -d /data/databases/pharokka
```

The Pharokka output is described [here](https://pharokka.readthedocs.io/en/latest/output/).
You get the annotation in standard formats such as ``gff`` or ``gbk``.

??? done "1. How many genes are identified and how many are protein coding?"
    80 genes, of which 79 are protein-coding (called CDS_0001 to CDS_0079) and one is a tRNA.

??? done "2. How many of the protein-coding genes have a functional prediction and for how many is the function unknown? How many have a PHROG assigned?"
    17 genes with a prediction and 62 with unknown function. 25 CDSs have a PHROG number.

You can also visualize the ``gbk`` using ``phold plot``:

```bash
phold plot \
    -i pharokka_524/pharokka.gbk \
    -o pharokka_plot
```

??? done "3. Can you confirm the number of genes with a functional annotation from the previous question?"
    Yes, we see 17 genes with an annotation.

Next, we try to improve the functional annotation using Phold.

**Note:** The following command takes 2-3 minutes. 
You can also continue with the results in ``/data/precomputed/virome/phold_524/``.

```bash
phold run \
    -i pharokka_524/pharokka.gbk \
    -o phold_524 \
    -t 4 \
    -d /data/databases/phold 
```

The output of Phold is described [here](https://github.com/gbouras13/phold#output).
You can also access the annotation in standard ``gbk`` format or as a tablular format (``phold_per_cds_predictions.tsv``).

??? done "4. Using Phold, how many genes have a PHROG assigned? How many have a functional prediction?"
    35 CDSs have a PHROG number, 30 genes have an annotation. We see that the annotation improved substantially for this contig.

Next, also visualize the phold ``gbk`` and compare the image to the one from pharokka:

```bash
phold plot \
    -i phold_524/phold.gbk \
    -o phold_plot
```

Finally, there is also an annotation inferred by genomad, you can find it in ``genomad/final.contigs_summary/final.contigs_virus_genes.tsv``

??? done "5. How many genes were annotated with genomad? How many of them have a functional prediction?"
    75 genes are found, 61 have no functional annotation (last column is NA), 14 have a function annotated.









