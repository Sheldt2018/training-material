---
layout: tutorial_hands_on
title: "Pre-processing of Single-Cell RNA Data"
zenodo_link: "https://zenodo.org/record/2573175"
tags:
  - single-cell
questions:
  - "What is single-cell?"
  - "How do we process a batch?"
  - "How do we process multiple batches?"
  - "How do we check for cross-contamination?"
objectives:
  - "Demultiplex FASTQ data via UMI-tools"
  - "Understand and validate the extraction of barcodes"
  - "Obtain an overview of general alignment and quantification techniques"
  - "Generate a count matrix for downstream single-cell RNA analysis"
time_estimation: "2h"
key_points:
  - "Verifying the distribution of barcodes via a FASTQC plot"
  - "Relocating barcodes into headers"
  - "Merging matrices from different batches together"
  - "Removing unwanted barcodes"
# requirements:
#   -
#     type: "internal"
#     topic_name: transcriptomics
#     tutorials:
#         - scrna-introduction
#         - scrna-plates-batches-barcodes
#         - scrna-umis
#
# follow_up_training:
#   -
#     type: "internal"
#     topic_name: transcriptomics

contributors:
  - mtekman
  - bebatut
  - erxleben
  - wm75

---

<!-- General Notes



-->


# Introduction
{:.no_toc}


#### Why do Single Cell sequencing?
{:.no_toc}

Single-cell RNA (scRNA) sequencing is the technological successor to classical "bulk" RNA-seq, where samples are no longer defined at the tissue level but at the individual cell level. The bulk RNA-seq methods seen in [previous hands-on material]({{site.baseurl}}{% link topics/transcriptomics/tutorials/ref-based/tutorial.md %}) would give the average expression of genes in a sample, whilst overlooking the distinct expression profiles given by the cell sub-populations due to their heterogeneity.

The rise of scRNA sequencing provides the means to explore this hetereogeneity by examining samples at the individual cell level, enabling a greater understanding of the development and function of such samples, by the characteristics of their constituent cells.

Consider the heterogenity of cells sampled from bone marrow, where hematopoietic stem cells can give rise to many different cell types within the same tissue:

![Cell Differentiation of Hematopoietic Stem cells]({{site.baseurl}}{% link topics/transcriptomics/images/scrna_hematopoiesis.png %} "Cell Differentiation of Hematopoietic Stem cells. Image Source: https://commons.wikimedia.org/wiki/File:Hematopoiesis_simple.svg")

The genes expressed by these cells at different developmental time points can be subtle, but generally can be classified into discrete cell subpopulations or under statistical clustering methods such as PCA or tSNE. Cells in the same cluster exhibit similar profiles of differential expression in the same set of related genes, compared to cells in other clusters. By identifying significant genes in each cluster, cell types and cell developmental heirarchies can be inferred based on the proximity of these clusters to one another.

Other than cell development, there are many more factors that can shape the level of gene expression exhibited by a given cell. Intercellular cell-signalling can block or enhance specific transcripts, the total amount of transcripts of a cell increases with the cell-cycle, or the proximity of a cell within a tissue to nutrients or oxygen.

![Facets of Cellular Identity]({{site.baseurl}}{% link topics/transcriptomics/images/scrna_nbt3711.png %} "Revealing the vectors of cellular identity with single-cell genomics, Nature Biotechnology, 2016")


This tutorial is in part inspired by aspects of the [Hemberg workflow](https://hemberg-lab.github.io/scRNA.seq.course/) at the Sanger institute, as well as the [CGATOxford workflow](https://github.com/CGATOxford/UMI-tools) which provides the **UMI-tools** suite that we make use of. The barcoding follows the [CEL-Seq2 protocol](https://doi.org/10.1186/s13059-016-0938-8) mentioned in the [*Understanding Barcodes*]({{ site.baseurl }}{% link topics/transcriptomics/tutorials/scrna-umis/tutorial.md %}) hands-on, and uses the same lane configuration as utilized by the [Freiburg MPI Grün lab](https://www.ie-freiburg.mpg.de/gruen).

# Analysis Strategy
{:.no_toc}

Most scRNA sequencing techniques use pooled-sequencing approaches to generate a higher throughput of data by performing amplification and sequencing upon multiple cells in the same "pool". From a bioinformatics standpoint, this means that the output FASTQ data from the sequencer is batch-specific and contains all the sequences from multiple cells, where one sample of cells is equal to one batch.

In this tutorial, we will perform pre-processing upon scRNA FASTQ batch data to generate an *N*-by-*M*  count matrix of *N* cells and *M* genes, with each element indicating the level of expression of that gene in a particular cell.

This count matrix is crucial for performing the downstream analysis, where differential gene analysis is performed between cells in order to cluster them into groups denoting their cell type and lineage. 

The tutorial is structured in two parts:

1. **Single-Batch Processing**
1. **Multi-Batch Processing**

![Overview of workflow]({{ site.baseurl }}{% link topics/transcriptomics/images/scrna_workflow.svg %} "An overview of the entire workflow")

The first part of this tutorial will deal with batches, and use example *FASTQ* data from a single batch, which we will perform [barcode extraction](#barcode-extraction) and annotation upon. Alignment and quality control will also be performed, and we will see how to construct a rudimentary count matrix.

The second part of this tutorial will deal with merging several output count matrices from multiple single batches generated in the first portion. Here, a set of example count matrices are [merged together](#merging-count-matrices) and quality control performed. This produces a final count matrix valid for downstream analysis.


> ### Agenda
>
> In this tutorial, we will deal with:
>
> 1. TOC
> {:toc}
>
{: .agenda}

# Single-Batch Processing

### Data upload and organization

In this tutorial we will be analysing scRNA-seq data of bone marrow cells taken from a single C57 mouse by *Herman et al.* ([2018](https://doi.org/10.1038/nmeth.4662)) and producing a count matrix that we can use for later analysis.

The size of scRNA files (.fastq) are typically in the gigabyte range and are somewhat impractical for training purposes, so we will expediate the analysis by using a smaller subset of actual batch data. We will also be using *Mus Musculus* annotation data (.gtf) from the  [NCBI RefSeq](ftp://ftp.ncbi.nlm.nih.gov/genomes/refseq/vertebrate_mammalian/Mus_musculus/) track, as well as a barcodes file (.tsv).


![Workflow Upper]({{site.baseurl}}{% link topics/transcriptomics/images/scrna_workflow_upper.png %} "Single batch processing")


> ### {% icon hands_on %} Hands-on: Data upload and organization
>
> 1. Create a new history and rename it (e.g. scRNA-seq single batch tutorial)
>
>    {% include snippets/history_create_new.md %}
>
> 1. Import the FASTQ paired data from [`Zenodo`](https://zenodo.org/record/2573175) or from the data library (ask your instructor)
>
>    {% include snippets/import_via_link.md collection=true collection_type="Paired" collection_name_convention="`<name>_<plate>_<batch>` to preserve the sample names, sequencing plate number and batch number." collection_name="Here we will write `C57_P1_B1`"  link="https://zenodo.org/record/2573175/files/SRR5683689_1.fastq.gz" link2="https://zenodo.org/record/2573175/files/SRR5683689_2.fastq.gz" genome="GRCm38/mm10" pairswaptext="`SRR5683689_1` and `SRR5683689_2`" %}
>
> 3. Import the Gene Annotations and Barcodes from [`Zenodo`](https://zenodo.org/record/2573175) or from the data library (ask your instructor)
>
>    ```
>    https://zenodo.org/record/2573175/files/Mus_musculus.GRCm38.93.mm10.UCSC.ncbiRefSeq.gtf
>    https://zenodo.org/record/2573175/files/celseq_barcodes.192.tabular
>    ```
>
> 4. Set the datatype of the `celseq_barcodes.192.tabular` to `tabular`
>
{: .hands_on}


For a more detailed understanding of the naming conventions used in generating our FASTQ pair, please consult the [*Plates, Batches, and Barcodes*]({{ site.baseurl }}{% link topics/transcriptomics/tutorials/scrna-plates-batches-barcodes/slides.html %}) slides.

## Barcode Extraction

It is highly recommended that you familiarise yourself with the concepts of barcodes and UMIs within multiplexed FASTQ data with the [*Understanding Barcodes*]({{site.baseurl}}{% link topics/transcriptomics/tutorials/scrna-umis/tutorial.md %}) hands-on material.

We will be performing barcode extraction on our batch data as given by the above hands-on tutorial, but this time filtering for *specific* cell barcodes, which we will ensure by providing the barcodes file.

> ### {% icon hands_on %} Hands-on: Barcode Extraction
>
> 1. **UMI-tools extract** {% icon tool %} with the following parameters:
>    - *"Library type"*: `Paired-end Dataset Collection`
>        - {% icon param-collection %} *"Reads in FASTQ format"*: `output` (Our paired set)
>        - *"Barcode on both reads?"*: `Barcode on first read only`
>    - *"Use Known Barcodes?"*: `Yes`
>        - {% icon param-file %} *"Barcode File"*: `output` (Input dataset)
>    - *"Barcode pattern for first read"*: `NNNNNNCCCCCC`
>    - *"Enable quality filter?"*: `No`
>
{: .hands_on}

As before, we can verify that the desired UMI and cell barcodes have been extracted from the sequence of the Forward reads and inserted into the header of the Reverse reads.

> ### {% icon question %} Question
> 
> 1. Why are input and output FASTQ file sizes so different?
> 2. How many reads were filtered out, and why?
>
> > ### {% icon solution %} Solution
> >
> > 1. The input FASTQ files contained reads from all barcodes, including those with sequencing errors, resulting in a larger pool of detected barcodes than those desired. (e.g. Cell barcode `AAATTT` could have single basepair sequencing errors that could modify it into `ATATTT` or `AAACTT`, etc).
> > 2. This information is included in the Log file of **UMI-tools extract** which contains all the parameters used to run, as well as *INFO* lines that indicate how many reads were read, and how many output. In this case: 14230244 reads (90.9%)
> {: .solution}
{: .question}


## Mapping / Alignment

FASTQ files contain sequence information that we wish to map to genes in a genome. Mapping is a relatively straightforward process, and is covered more extensively in the [*Sequence Analysis*]({{site.baseurl}}{% link topics/sequence-analysis/index.md %}) tutorials:

1. Select your genome
1. Select your gene annotation file
1. Run the alignment program
1. (Optional) Run MultiQC to assess the quality of the mapping

The FASTQ data was sequenced from mouse data, so to perform the alignment we will need to gather all data relevant to that genome. We will use the latest version (mm10).

The annotation GTF file must match the genome version used, since both use physical coordinates. Each GTF contains all the gene, exon, intron, and other regions of interest that we will use to annotate our reads, should our reads align to any of the regions specified in this file.

For alignment, we will use RNA-STAR for performance and splice-awareness.


> ### {% icon hands_on %} Hands-on: Performing the Alignment
>
> 1. **RNA-STAR** {%icon tool %} with the following parameters:
>    - *"Single-end or paired-end reads"*: `Single-end`
>        - {% icon param-file %} *"RNA-Seq FASTQ/FASTA file"*: `out2` (output of **UMI-tools extract** {% icon tool %})
>    - *"Custom or built-in reference genome"*: `Use a built-in index`
>        - *"Reference genome with or without an annotation"*: `use genome reference without builtin gene-model`
>            - {% icon param-file %} *"Select reference genome"*: `Mus Musculus (mm10)` (Mouse)
>            - {% icon param-file %} *"Gene model (gff3,gtf) file for splice junctions"*: `Mus_musculus.GRCm38.93.mm10.UCSC.ncbiRefSeq`
> 1. **MultiQC** {%icon tool %} with the following parameters:
>    - *"Results"*:
>      - *"1: Results"*:
>        - *"Which tool was used to generate logs?"*:`STAR`
>        - *"STAR output"*:
>           - *"1: STAR output"*
>              - *"Type of STAR output?"*:`Log`
>        - *"STAR log output"* :(Select the STAR output file that ends in "log")
> 1. Click on the {% icon galaxy-eye %} symbol on the MultiQC Webpage
>
{: .hands_on}

The purpose of MultiQC is to observe how well our reads were mapped against the reference genome. Many reads are discarded due to being of too low quality, or having ambiguous sequence content that can map them to multiple locations.

> ### {% icon question %} Question
> 
> 1. What percentage of our reads are uniquely mapped? How many millions of reads is this percentage?
> 2. What percentage of our reads are mapped to more than one locus?
> 3. Is our overall mapping 'good' ?
>
> > ### {% icon solution %} Solution
> > 
> > 1. `70.9%` or 10 million reads were successfully mapped
> > 2. `14.4%` are multiply mapped, and `2.2%` were mapped to too many loci
> >   - Multiply mapped means that a read was aligned to more than one gene
> >   - Mapped to too many loci means that a read was aligned to 10 or more loci, and should be ignored.
> > 3. It depends on how good the sequencing protocol is, and how many reads in total were mapped.
> >   - `90%` is amazing, reserved for bulk RNA-seq which typically has high coverage
> >   - `70%` is weak for bulk RNA-seq, but good for single-cell RNA-seq
> >   - 6 million mapped reads should be enough to generate a downstream analysis from.
> >
> {: .solution}
{: .question}


## Filtering

Before continuing let us first look back on some of the previous stages:

> ### {% icon comment %} Recap of previous stages
>
> 1. *Barcode Extraction*:  
>
>     Here we used `umi_tools extract` on our input forward and reverse FASTQ files, and extracted the UMI and cell barcode from the forward read *sequence*, and placed it into the *header* of both forward and reverse FASTQ files. i.e. FASTQ files → Modified FASTQ files  
>
> 2. *Mapping*:  
>
>     We took the sequencing data from the reverse FASTQ file (with modified headers) and aligned it to the mouse genome, using annotations presented in the GTF file for that genome. i.e. Modified FASTQ file (reverse) → BAM file
>
{: .comment}

#### Confirming Reads in the BAM file

We now have a BAM file of our aligned reads, with cell and UMI barcodes embedded in the read headers. We also have the chromosome and base-pair positions of where these reads are aligned. The can be confirmed by peeking into the BAM file:

> ### {% icon hands_on %} Hands-on: Confirming the Alignment Data
>
>  1. Click on the {% icon galaxy-eye %} symbol of the BAM output from STAR.
>  2. There are many header lines that begin with `@` which we are not interested in. 
>  3. Look at first read directly below the header lines:
> 
>         J00182:75:HTKJNBBXX:2:1121:9729:45889_GACGAA_GTGGTC	16	chr1	2030	3	70M	*	0	0	AGAGGTTCCAATATTCCCATGAAATTGAGATTTTGTAAAAGAGTGAAGTGTGGTTACTTTCACTGAGAGG	JJJJJJJJJJJJJJJJJJJJJJJJFJJJJJAJJJJJJJJJJFJFJFFJJJJJJJJJJJJFF7AJA-77<A	NH:i:2 HI:i:1 AS:i:64 nM:i:2
>
{: .hands_on}

The fields of the BAM file can be better explained at section 1.4 of [the SAM specification](https://samtools.github.io/hts-specs/SAMv1.pdf), but we will summarize the main fields of interest here:

* `J00182..._GACGAA_GCGGTC`: The *readname* appended by `_`, the cell barcode, another `_`, and then the UMI barcode.
* `16`: The FLAG value

> ### {% icon question %} What does the FLAG value of 16 tell us about this read?
>
>   <!-- TODO This information needs to be integrated into an actual tool -->
>   We can interactively see what the different FLAG values mean by feeding values into the SAM specification to the [Picard web tool](https://broadinstitute.github.io/picard/explain-flags.html)
>
>   > ### {% icon solution %} Solution
>   > The read aligns to the reverse strand
>   >
>   {: .solution}
>
{: .question}

* `chr1` `2030`: The position and base-pair of alignment of the first base of the sequence.
* A series of quality fields, with the main contributors being  the sequence and sequence quality strings.
* `NH`: The number of hits for  this read. If it is multiply mapped, then the number of multiples will be shown (here `2`).
* `HI`: Which number this particular read is in the series of (potentially) multi-mapped reads (here `1`, not neccesarily meaning the first or 'better').
* `nM`: The number of mismatches in the alignment of this read to the reference (here `2`).


#### Filtering the BAM File

If we perform counting on the current BAM file we will be counting all reads, even the undesireable ones such as those that did not align so optimally.

The main filtering steps performed on our reads so far have been relatively silent due to the 'default' parameters used.
* *UMI-tools Extract* - Filters reads for those only with matching barcodes given by our barcodes file.
* *RNA-STAR* - As seen in the log, we lose 10% of our reads for being too short or being multiply mapped.

Another filtering measure we can apply is to keep reads that we are confident about, e.g those with a minimum number of mismatches to the reference within an acceptable range. 

> ### {% icon hands_on %} Hands-on: Task description
>
> 1. **Filter BAM datasets on a variety of attributes** {% icon tool %} with the following parameters:
>    - {% icon param-file %} *"BAM dataset(s) to filter"*: `output_bam` (output of **RNA STAR** {% icon tool %})
>    - In *"Condition"*:
>        - In *"1: Condition"*:
>            - In *"Filter"*:
>                - In *"1: Filter"*:
>                    - *"Select BAM property to filter on"*: `alignmentFlag`
>                        - *"Filter on this alignment flag"*: `0`
>        - Click on *"Insert Condition"*:
>        - In *"2: Condition"*:
>            - In *"Filter"*:
>                - In *"1: Filter"*:
>                    - *"Select BAM property to filter on"*: `alignmentFlag`
>                        - *"Filter on this alignment flag"*: `16`
>        - Click on *"Insert Condition"*:
>        - In *"3: Condition"*:
>            - In *"Filter"*:
>                - In *"1: Filter"*:
>                    - *"Select BAM property to filter on"*: `tag`
>                        - *"Filter on a particular tag"*: `nM:<3`  
>                          <small>(**Attention! please use a lowercase 'n' here!**)  
>        - Click on *"Insert Condition"*:
>        - In *"4: Condition"*:
>            - In *"Filter"*:
>                - In *"1: Filter"*:
>                    - *"Select BAM property to filter on"*: `tag`
>                        - *"Filter on a particular tag"*: `NH:<2`
>    - *"Would you like to set rules?"*: `Yes`
>        - *"Enter rules here"*: `(1 | 2) & 3 & 4`
>
{: .hands_on}


## Quantification 

Once we have the name of the gene for a specific read, we can count how many of those reads fall into that gene and generate a count matrix.

Counting reads is performed by two commonly-used tools:
1. RNA-STAR
2. FeatureCounts

The **RNA-STAR** tools has the ability to count reads as it maps them. **FeatureCounts** performs the same task, but is capable of counting not just at the Read level, but also at the UMI level too, such that 10 duplicate reads will be counted only once.

Unfortunately, both are currently limited to counting without being able to distinguish between different cells.

> ### {% icon details %} Example
>
> If we consider the number of reads that align to *GeneA*, the output given by these two tools is as follows:
>
> > | (reads) | RNA STAR | FeatureCounts |
> > |---------|----------|---------------|
> > | GeneA   |   12     |    12         |
> {: .matrix}
>
> But what we actually require is:
>
> > | (reads) | C1 | C2 |
> > |---------|----|----|
> > | GeneA   | 10 |  2 |
> {: .matrix}
>
> or more specifically:
>
> > | (UMIs) | C1 | C2 |
> > |--------|----|----|
> > | GeneA  | 2  |  1 |
> {: .matrix}
>
{: .details}


In order to obtain this desired format, we must use **UMI-tools count** to perform the counting. However, this tool is dependent on **FeatureCounts** to annotate our reads with the one crucial piece of information that is missing from our BAM file: the name of the gene.

> ### {% icon tip %} Tip: Verifying missing gene name
> You can check this yourself by examining the {% icon galaxy-eye %} of the BAM file *"STAR Alignment file"*
{: .comment}

### Annotating Gene name with FeatureCounts

Let us annotate our BAM file with desired gene tags.

> ### {% icon hands_on %} Hands-on: Quantification assist via FeatureCounts
> 
> 1. **FeatureCounts** {%icon tool %} with the following parameters:
>    - {% icon param-file %} *"Alignment file"*: `mapped_reads` (output of **RNA STAR** {% icon tool %})
>    - *"Gene annotation file"*: `in your history`
>        - {% icon param-file %} *"Gene annotation file"*: `Mus_musculus.GRCm38.93.mm10.UCSC.ncbiRefSeq.gtf`
>    - *"Specify strand information"*:`Unstranded`
>    - In *"Advanced options"*:
>        - *"Count multi-mapping reads/fragments"*: `Disabled; multi-mapping reads are excluded (default)`
>        - *"Exon-exon junctions"*: `Yes`
>        - *"Annotates the alignment file with 'XS:Z:'-tags to described per read or read-pair the corresponding assigned feature(s)."*: `Yes`
>
> 2. Examine the output BAM file
>  - Click on the {% icon galaxy-eye %} for the "Feature Counts: Alignment File"
>  - Scroll down past the header lines
>  - Scroll horizontally to the tags, observe the new `XT` tag.
{: .hands_on}

The `XS` and `XT` tags in the BAM file will now form the basis for counting reads.
With all the relevant data now in our BAM file, we can actually perform the counting via `UMI-tools count`.

> ### {% icon tip %} Tip: Verifying added gene name
> You can once again check this yourself by examining the {% icon galaxy-eye %} of the BAM file *"STAR Alignment file"*
{: .comment}


### Counting Genes / Cell

> ### {% icon hands_on %} Hands-on: Final Quantification
> 
> 1. **UMI-tools counts** {% icon tool %} with the following parameters:
>    - {% icon param-file %} *"Sorted BAM file"*: `out_file1` (output of **FeatureCounts** {% icon tool %})
>    - *"UMI Extract Method"*: `Barcodes are contained at the end of the read seperated by a delimiter`
>    - *"Bam is paired-end"*:`No`
>    - *"Method to identify group of reads"*:`Unique`
>    - *"Extra Parameters"*:
>       - *"Deduplicate per gene."*:`XT`
>       - *"Group reads only if they have the same cell barcode."*:`Yes`
>    - *"Prepend a label to all column headers"*:`No modifications`
>
{: .hands_on}


The important parameters to take note of are those given in the *Extra Parameters* where we specify that each of the reads with a `XT:Z` tag in the BAM file will be counted on a per cell basis. Reads sharing the same UMI and cell Barcodes will be de-duplicated into a single count, reducing PCR duplicate bias from the analysis.

At this stage, we now have a tabular file containing genes/features as rows, and cell labels as headers.

> ### {% icon question %} Question
> 
> 1. How many genes do we have in the matrix?
> 2. How many cells?
>
> > ### {% icon solution %} Solution
> >
> > 1. ~23,000 lines  
> >    This information can be seen in the file preview window by clicking on the name of the file (**NOT** the {% icon galaxy-eye %} symbol). 
> >
> > 2. 192 columns (not including the first column of gene names)  
> >    The number of columns can be seen by scrolling the file preview window completely to the right.
> >
> {: .solution}
{: .question}

The generation of a single count matrix is now complete, with the emphasis on the word *single* due to the fact that we often deal in multiple batches of sequencing data.


> ### {% icon comment %} Recap of previous stages
>
> 1. *Barcode Extraction*:  
>   
>     Here we used `UMI_tools extract` on our input forward and reverse FASTQ files, and extracted the UMI and cell barcode from the forward read *sequence*, and placed it into the *header* of both forward and reverse FASTQ files. i.e. FASTQ files → Modified FASTQ files  
> 2. *Mapping*:
>
>     We took the sequencing data from the reverse FASTQ file (with modified headers) and aligned it to the mouse genome, using annotations presented in the GTF file for that genome. i.e. Modified FASTQ file (reverse) → BAM file
>
> 3. *Quality Filtering*:  
>
>     Reads with aligment mismatches greater than 2 were discarded, and only non multi-mapped reads that mapped to the forward or reverse strand were kept
>
> 4. *Quantification*:  
>
>     Gene tags were added to our alignment file, and reads were grouped according those sharing the same gene tag, with further reduction performed by collapsing all reads sharing the same cell and UMI barcode to be counted only once.
>
{: .comment}

This concludes the first part of the tutorial which focused on the transformation of raw FASTQ data from a single batch into a count matrix. The second part of this tutorial guides us through the process of merging multiple processed batches from the first stage, and performing qualitative filtering.


> ### {% icon details %} Workflows For All Steps
>
> Galaxy provides a workflow that captures the process of all the above stages for a [single pair of FASTQ data](workflows/scrna_pp_celseq.ga):
>
> ![Single Batch Workflow]({{site.baseurl}}/topics/transcriptomics/images/scrna_workflow_single.png "Single Batch Workflow")
>
> For multiple batch processing, Galaxy can make use of *Nested Workflows* that in this case can take in a [list of input paired FASTQ data](workflows/scrna_mp_celseq.ga) and process them in parallel.
>
> ![Multiple Batch Workflow]({{site.baseurl}}/topics/transcriptomics/images/scrna_workflow_nested.png "Nested Multiple Batch Workflow")
>
{: .details}


# Multi-Batch Processing


![Workflow Lower]({{site.baseurl}}{% link topics/transcriptomics/images/scrna_workflow_lower.png %} "Multiple batch processing")

Handling more than one batch of sequencing data is rather trivial when we take into account our main goals and requirements:

1. For each batch, convert FASTQ reads from into a count matrix.
2. Merge all count matrices into a single count matrix

The first step merely requires us to run the same workflow on each of our batches, using the exact same inputs except for the FASTQ paired data. The second step requires a minimal level of interaction from us; namely using a merge tool and selecting our matrices. 


### Data upload and organization

The count matrix we have generated in the previous section is too sparse to perform any reasonable analysis upon, and constitutes data only of a single batch. Here we will use more populated count matrices from multiple batches, under the assumption that we now know how to generate each individual one of them using the steps provided in the previous section. This data is available at [`Zenodo`](https://zenodo.org/record/2573175).

Once again, file naming is important, and so we will rename our matrix files appropriately to the plate and batch they are supposed to originate from.

> ### {% icon hands_on %} Hands-on: Data upload and organization
>
> 1. Create a new history and rename it (*e.g.* scRNA-seq multiple-batch tutorial)
> 1. Import the four matrices (`P1_B1.tabular`, `P1_B2.tabular`, etc.) from [`Zenodo`](https://zenodo.org/record/2573175) or from the data library (ask your instructor)
>    - Set the datatype of the tabular files to **tabular**
>
>    ```
>    https://zenodo.org/record/2573175/files/P1_B1.tabular
>    https://zenodo.org/record/2573175/files/P1_B2.tabular
>    https://zenodo.org/record/2573175/files/P2_B3.tabular
>    https://zenodo.org/record/2573175/files/P2_B4.tabular
>    ```
>    
>    {% include snippets/import_via_link.md %}
>    {% include snippets/import_from_data_library.md %}
>    
> 1. Rename a matrix
>    - Click on {% icon galaxy-pencil %} of the *`P1_B1.tabular`* file
>    - Set the Name field such that it is affixed with "_P1_B1" (e.g. 'multibatch_P1_B1')
>    - Click *Save*
> 1. Repeat for all matrices
>    **Pay attention to the Plate number which changes after Batch 2**
>
{: .hands_on}

## Merging Count Matrices

Before we begin, we must consider that our matrices are not equal -- e.g. Batch1 has 3 cells that describe *Genes{A,B,C,D}* whereas Batch2 has 4 cells that describe *Genes{C,D,E}*.

We have the problem that only *GeneC* and *GeneD* appear in both batches, whilst describing 7 different cells in total.

To resolve this we can perform a "Full Table Join" where the missing data for *GeneE* and *GeneA* in Batch1 and Batch2 respectively are replaced with zeroes:

![Table Join]({{site.baseurl}}{% link topics/transcriptomics/images/scrna_fulltable.svg %} "Full Table Join")

> ### {% icon comment %} Comments
> For more information on table joins, see [here](http://www.sql-join.com/sql-join-types/)
{: .comment}


> ### {% icon question %} Question
>
> 1. Why have the column headers changed in the Full matrix?
> 2. Why are the cell labels in B1 and B2 the same, if they are labelling completely different cells?
> 
> > ### {% icon solution %} Solution
> >
> > 1. Although the cell headers in each batch matrix is the same, the cells they label are *not* the same and need to be relabelled in the final matrix to tell us which batch they originated from.
> > 2. The reason the cell headers are the same is because the cells use the same barcodes, due to fact that the *same* barcodes are sometimes used across *different* batches.
> >
> {: .solution}
> 
{: .question}


Let us now merge our matrices from different batches.

> ### {% icon hands_on %} Hands-on: Table Merge
> 
> 1. **Column Join on Collections** {% icon tool %} with the following parameters:
>   - *"Tabular Files"*: (Select each of the matrices that you wish to join)  
>   - *"Identifier column"*:`1`  
>   - *"Number of Header lines in each item"*:`1`  
>   - *"Keep original column header"*:`Yes`  
>   - *"Fill character"*:`0`  
>
{: .hands_on}

The identifier column refers to the column where the gene names are listed. A 1:1 correspondence between matrices is checked, so that the merge does not concatenate the wrong rows between matrices. The *Fill character* provides a default value of 0 for cases where a Gene appears only in one of the matrices as per our example earlier.

Once the merge is complete, we can now peek at our full combined matrix by once again clicking on the file name to see a small summary. Here we can see that we now have ~30,000 genes and over 1500 cells.

However, the number of cells are greatly overestimated.  This is because *not all batches use the same barcodes*, and yet we are applying the full set of barcodes to each batch.

The reason we do this is to test for cross-contamination between batches, the details of which are better explained in the [*Plates, Batches, and Barcodes*]({{ site.baseurl }}{% link topics/transcriptomics/tutorials/scrna-plates-batches-barcodes/slides.html %}) slides.


## Guarding against Cross-Contamination

There are multiple possible ways to configure a plate for sequencing multiple batches. Thankfully, Galaxy provides a tool that caters for this, and checks for cross-contamination in any experimental setup. It requires only the following information:

1. A full list of barcodes
2. Which barcodes apply to which batches
3. Which batches apply to which plates.
 
Since we the plating protocol we are using is that designed by the Freiburg MPI Grün lab, we will follow their structure.

> ### {% icon details %} Details: Plating protocol
> 
> - Barcodes:
>     These are each 8bp long, with an edit distance of 2, and there 192 of them.
> 
>     > |:-------:|:-----------:|
>     > | 001-006 | AACACC AACCTC AACGAG AACTGG AAGCAC AAGCCA |
>     > | 007-012 | AAGGTG AAGTGC ACAAGC ACAGAC ACAGGA ACAGTG |
>     > |    .    |        .  |
>     > |    .    |        .  |
>     > | 180-186 | TTACGC TTCACC TTCCAG TTCGAC TTCTCG TTGCAC |
>     > | 187-192 | TTGCGA TTGCTG TTGGAG TTGGCA TTGGTC TTGTGC |
>     {: .matrix}
>
> - Plates:
>     Here we have 8 batches spread out over 2 plates, with alternate barcode striping.
>           
>     > |          | 001-096 | 097-192 | 001-096 | 097-192 |
>     > |----------+---------+---------+---------+---------|
>     > | Plate 1  |  B1     | B2      | B3      | B4      |
>     > | Plate 2  |  B5     | B6      | B7      | B8      |
>     {: .matrix}
>
{: .details}

This plating protocol can be converted into a more textual format, which allows for many variable setups (see *Help* section of *Cross-contamination Barcode Filter* tool).

    [Barcodes → Batches]
    001-096: B1 , B3 , B5 , B7
    097-192: B2 , B2 , B6 , B8

    [Plates → Batches]
    1: B1 , B2 , B3 , B4
    2: B5 , B6 , B7 , B8

Let us now apply this protocol to our count matrix, and look for any cross-contamination.

> ### {% icon hands_on %} Hands-on: Barcode Filtering
> 
> Select **Cross-contamination Barcode Filter** {%icon tool %} with the following parameters:
>  - *"Input Matrix"*:`output` (merged matrices from the Column Join tool)
>  - *"Complete Barcodes"*:(barcodes file)
>  - *"Plate Protocol"*:`Custom`
>     - *"Under 'Barcode Format'"*:
>        - Select `+ Insert Barcode Format`:
>           - *"1: Barcode Format"*:
>              - *"Barcode Range: Start"*:`1`
>              - *"Barcode Range: End"*:`96`
>              - *"Batches utilizing this Range"*:`2,4`
>        - Select `+ Insert Barcode Format`:
>           - *"2: Barcode Format"*:
>              - *"Barcode Range: Start"*:`97`
>              - *"Barcode Range: End"*:`192`
>              - *"Batches utilizing this Range"*:`1,3`
>     - *"Under 'Plate Format'"*:
>        - Select `+ Insert Plate Format`:
>           - *"1: Plate Format"*:
>              - *"Plate Number"*:`1`
>              - *"Batches within this Plate Number"*:`1,2`
>        - Select `+ Insert Plate Format`:
>           - *"2: Plate Format"*:
>              - *"Plate Number"*:`2`
>              - *"Batches within this Plate Number"*:`3,4`
>  - Expand the *"RegEx Parameters"* section:
>     - *"RegEx to extract Plate, Batch, and Barcodes from headers"*:`.*P(\\d)_B(\\d)_([ACTG]+)`
>       <small>(**Attention! Take note of the 'B'**)
>
{: .hands_on}

The [regular expression](https://www.regular-expressions.info/quickstart.html) (RegEx) used in that final step is required to tell us how to capture the important information in the cell headers contained in brackets `(` `)`, where `\\d` denotes an expected digit, and `[ACTG]+` denotes 1 or more characters matching A or C or T or G.
 
The plot that follows tells us everything we need to know about each of our batches. Each batch is essentially tested against the full set of barcodes in order to assert that only the desired or 'Real' barcodes have been sequenced.

#### Cross-contamination Plot

![Contamination Plots]({{site.baseurl}}{% link topics/transcriptomics/images/scrna_crosscontamination.png %} "The Pre-filter and Post-filter plots")

1. In the pre-filter plot, we can see how only half of the sequences in each batch map to half the barcodes. This shows very little cross-contamination, and proves that our data is real.
2. The post-filter plot essentially removes the false barcodes from each batch and retains only the 'Real' barcodes.

> ### {% icon question %} Question
>
> The count matrix that is output from this tool has only half the number of cells as the original input count matrix. Why is this?
>
> > ### {% icon solution %} Solution
> >
> > Because only half the barcodes in each batch were real. The *UMI-tools extract* took the entire barcodes file to filter against each batch, and the *UMI-tools count* also took the entire barcodes file to count against each batch. 
> >
> > Naturally, each batch produced 192 cells, even though 96 were real. As a result of joining each of these matrices we ended up with a count-matrix of $$8 * 192 = 1536$$ cells. The cross-contamination tool removes the false barcodes (50% in each batch), resulting in $$768$$ cells.
> >
> {: .solution}
{: .question}

With this, we now have a count-matrix that can be used for further downstream analyis. 

<!-- 
Factoid: We can convert the number of UMIs to the number of molecules using a transformation script.
-->
 
# Conclusion
{:.no_toc}

In this tutorial we have learned the importance of barcoding; namely how to define, extract, and annotate them from our reads and into the read headers, in order to preserve them during mapping. We have discovered how these barcoded reads are transformed into counts, where the cell barcode and UMI barcode are used to denote individual cells and to correct against reads that are PCR duplicates. Finally, we have learned how to combine seperate batch data as well as being able to check and correct against cross-contamination.

![Recap of workflow]({{site.baseurl}}{% link topics/transcriptomics/images/scrna_workflow.svg %} "A recap of the entire workflow")

