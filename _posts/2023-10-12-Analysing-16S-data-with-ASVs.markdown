---
layout: post
title:  "02 Analysing 16S data with ASVs"
date:   2023-10-10 09:00:01
categories: jekyll update
---


# Introduction

The goal of this practical session is to evaluate using a denoising approach as an
alternative to OTU calling for removal of errors in 16S sequence data. A number of
tools have beend developed for this purpose (e.g. [Minimum entropy decomposition][med],
[UNOISE][unoise], and [DEBLUR][deblur]). This practical us based around the use of [DADA2][dada2]).


<br>

&nbsp;&nbsp;[1. Fetching Data for DADA2 Analysis](#header1) <br>
&nbsp;&nbsp;[2. Open DADA2 and Plot Read Quality](#header2) <br>
&nbsp;&nbsp;[3. Trim and filter forward and reverse reads based on quality](#header3) <br>
&nbsp;&nbsp;[4. Denoise reads with DADA2](#header4) <br>
&nbsp;&nbsp;[5. Genererate amplicons and create an abundance matrix](#header5) <br>
&nbsp;&nbsp;[6. Assign Taxonomy to DADA2 Amplicon Sequence Variants](#header6) <br>

<br>

&nbsp;&nbsp;[Conlusions](#header7) <br>

<br>
<br>
<br>


----------------------------------
# 1. Fetching Data for DADA2 Analysis <a name="header1"></a>

We'll begin by opening a terminal, then creating a new working directory for this session.
{% highlight bash %}
cd ~
# Create working directory
mkdir amplicon_asv
cd amplicon_asv
# Fetch input fastq files containing amplicon sequence data
mkdir fastqs
cd fastqs
cp  ~/.thursday_pm/data/input_fastq/*fastq.*.gz .
{% endhighlight %}

Note that there are now gzip-compressed paired end sequence data for three samples
in this location.

{% highlight bash %}
ls -l

# Files are gzip compressed, but it is still possible to view them
# using the linux pipe.
zcat sample1_trimmed.fastq.1.gz | head -4

# Move back into our working directory
cd ..
{% endhighlight %}

<br>
<br>

----------------------------------
# 2. Open DADA2 and Plot Read Quality <a name="header2"></a>

As the DADA2 package is written in R, most of this session will be
carried out in RStudio.

Open RStudio from your linux desktop and open a new R script.

![RStudio]({{ site.baseurl }}/images/new_rscript.png)

Remember, you can write (and comment!) code in your new R script then run it via
the R terminal, either by copying and pasting it across, or by highlighting 
it and selecting the `Run` icon in the top-right side of your script window.

![RStudio]({{ site.baseurl }}/images/rstudio_run.png)

<br>

Having opened RStudio, set your working directory to the location of
your run directory and load the Dada2 library.

{% highlight R %}
# Find out which working directory R is running in? 
getwd()

# Set your working directory and check the contents
setwd("~/amplicon_asv/")
list.files('.')
list.files('./fastqs')

# Load Dada2
library('dada2')

{% endhighlight %}


Note that these fastq files have been trimmed to remove PCR and sequencing primers.
However, they have not been trimmed to remove low quality sequences at the 3'
end of reads. We will use the DADA2 function `plotQualityProfile()` to visualize
the distribution in quality at each base in the forward and reverse reads from
the first sample. 

{% highlight R %}
# Create vectors in R containing details of forward and reverse reads
fastq1 <- sort(list.files('fastqs', pattern='.fastq.1.gz', full.names=T))
fastq2 <- sort(list.files('fastqs', pattern='.fastq.2.gz', full.names=T))

# Plot the read quality distribution for each sample pair
plotQualityProfile(c(fastq1, fastq2))

{% endhighlight %}

![RStudio]({{ site.baseurl }}/images/dada2_quality_plot.png)

In this plot lines show the mean (green), median (orange) and 25th/75th 
percentiles (dashed) of the quality score at each base position in the forward
and reverse reads.


<br>
<br>

----------------------------------
# 3. Trim and filter forward and reverse reads based on quality<a name="header3"></a>

The first step in processing reads with DADA2 is to remove poor quality sequence.
(Note in the [OTU tutorial]({{ site.baseurl }}{% post_url 2023-10-12-Analysing-16S-data-with-OTUs %})
we were relying on the FLASh assembler and downstream OTU calling to account for low quality sequence.) 

Similar to working on the commandline, it's necessary to define the names of input and output
files before running the filtering step. 

{% highlight R %}
# Create a subdirectory from within R to contain quality trimmed sequences
dir.create('fastqs_trimmed')

# This subdirectory now appears in your working directory
list.files('.')
list.files('./fastqs_trimmed')

# Fetch a list of sample names from the list of forward reads
sample.names <- sapply(strsplit(basename(fastq1), '_'), `[`, 1)

# Create vectors of output filenames for both forward and reverse reads
fastq1.filtered <- file.path('fastqs_trimmed', paste0(sample.names, '.fastq.1.gz'))
fastq2.filtered <- file.path('fastqs_trimmed', paste0(sample.names, '.fastq.2.gz'))

# Run quality trimming and filtering in DADA2
out <- filterAndTrim(fwd=fastq1, filt=fastq1.filtered, 
       		     rev=fastq2, filt.rev=fastq2.filtered,
                     truncLen=c(260, 260), 
		     maxEE=c(2,2),
                     multithread=T)

# The subdirectory should now contain quality trimmed reads
list.files('./fastqs_trimmed')
print(out)

# Have a closer look at the arguments supplied to the filterAndTrim function
?filterAndTrim

{% endhighlight %}

In the code above, we've set the minimum permissible truncated length of for a
sequence to be 260bp for both forward and reverse reads. These were originally
2x300bp reads generated on the Illumina MiSeq platform to span the (~500bp) V1-V3
region. However, 20bp has been trimmed from the 5' end of each read during adapter
removal. This minimum lengths used above have therefore been selected to ensure (minimal!) 
overlap between reads during amplicon assembly. 

The histogram below shows a typical distribution in the length of V1-V3 amplicons generated 
from a human gut microbiome study.

![RStudio]({{ site.baseurl }}/images/histogram.png)

> What biases may be introduced by setting a stringent `truncLen` value?

> For more information on the arguments supplied in the quality trimming and filtering
step type `?filterAndTrim` into the R terminal.

> Try altering `truncLen` and replacing `maxEE` with a minimum quality score (`minQ`)
to see what effect these parameters have on the number of reads remaining after filtering.

<br>
<br>

----------------------------------
# 4. Denoise reads with DADA2<a name="header4"></a>

Denoising  in DADA2 is a two step process, first it models the expected error in the 
forward and reverse reads, then it removes this error to create a corrected read set.
For further detail see the methods and supplementary material in the original 
[DADA2 publication][dada2].

{% highlight R %}
# Model error in forward and reverse reads
error1 <- learnErrors(fastq1.filtered)
error2 <- learnErrors(fastq2.filtered)

# Remove exact replicate sequences
dereplicated1 <- derepFastq(fastq1.filtered, verbose=T)
names(dereplicated1) <- sample.names
dereplicated2 <- derepFastq(fastq2.filtered, verbose=T)
names(dereplicated2) <- sample.names

# Then apply the DADA2 denoising algorithm
corrected1 <- dada(dereplicated1, err=error1)
corrected2 <- dada(dereplicated2, err=error2)

corrected1
corrected2
{% endhighlight %}

<br>
<br>

----------------------------------
# 5. Genererate amplicons and create an abundance matrix<a name="header5"></a>

Having corrected sequencing errors in forward and reverse reads, the remaining
steps are to merge read pairs into amplicons, remove chimeras, and to calculate
the abundance of each amplicon in each sample. 

{% highlight R %}
# Combine the forward and reverse read into a single set of amplicons
amplicons <- mergePairs(dadaF=corrected1, derepF=dereplicated1,
                        dadaR=corrected2, derepR=dereplicated2,
                        verbose=T)

# Generate a count matrix containing the counts for each denoised sequence
sequence.tab <- makeSequenceTable(amplicons, orderBy='abundance')

# The rows of this matrix are the samples
rownames(sequence.tab)
# The columns are the sequences
colnames(sequence.tab)

# Remove chimeras directly from the count.matrix
sequence.tab.nochimeras <- removeBimeraDenovo(sequence.tab, verbose=T)

# Finally, it's possible to write the denoised amplicons to a fasta file
uniquesToFasta(getUniques(sequence.tab.nochimeras), 'dada2_asvs.fasta')

{% endhighlight %}

> How many unique amplicons were created using the DADA2 denoising pipeline?

> How many sequences were lost compared to using the FLASh assembler?


<br>
<br>

----------------------------------
# 6. Assign Taxonomy to DADA2 Amplicon Sequence Variants<a name="header6"></a>

Within the DADA2 pipeline, it is possible to assign taxonomy to denoised
sequences via a two step process. The first step is to use the `assignTaxonomy()`
function. (Use `?assignTaxonomy` to see how this compares to the method used in 
the accompanying [OTU tutorial]({{ site.baseurl }}{% post_url 2023-10-12-Analysing-16S-data-with-OTUs %})).
As this function only returns classifications to genus level, the second step `addSpecies()`
attempts to find exact matchesfor each sequence in a curated reference database. 

First, it's necessary to retrieve reference databases for taxonomic and species
assignment. These have been downloaded for you and can be copied directly into your
current working directory.


{% highlight bash %}
# In your bash terminal cd into the current working directory
cd ~/amplicon_asv

# Copy the required reference fasta files to your working directory
cp ~/.thursday_pm/data/GTDBr95* .
ls -l
{% endhighlight %}

Having retrieved the reference databases, return to your RStudio session to perform
taxonomic assignment.

{% highlight R %}
# Assign taxonomy to genus level
taxonomy.table <- assignTaxonomy(seqs=sequence.tab.nochimeras,
                       refFasta="GTDBr95-Genus.fna",
                       multithread=T)
# Assign species via exact matching
taxonomy.table <- addSpecies(taxtab=taxonomy.table, 
                             refFasta="GTDBr95-Species.fna", 
                             verbose=T)

# As row names in the resulting table are the sequences
# remove them before viewing.
taxa.print <- taxonomy.table
row.names(taxa.print) <- NULL
taxa.print <- as.data.frame(taxa.print)
View(taxa.print)
{% endhighlight %}

<br>

Alternatively, it is possible to assign taxonomy to the denoised amplicon 
sequences by using the NCBI nucleotide BLAST tool. This can be done by copying and
pasting individual sequences into the [online BLAST portal][blast]. Then selecting 
the highly-curated NCBI 16S rRNA database as a reference. 


![BLAST]({{ site.baseurl }}/images/blast_image.png)


>Try running BLAST for the most and least abundant ASVs in your output fasta file.

>Try running BLAST against the NCBI Nucleotide collection database, rather than the
16S rRNA database. 


Note it's  also possible to install BLAST locally and run it from  the
[commandline][blastcmd]. However, if you do this for taxonomic assignment, 
[be careful about the arguments you provide][max_target_seqs].

<br>
<br>

## Conclusions<a name="header7"></a> 


Biological judgement's required when interpreting both OTU and denoised amplicon data.
In this example, all three samples were generated from sequencing the same mock community
that (when put together) contained 36 species and 24 genera.

A table of the taxa present in the sample is given below:


| | |
|----|----|----|
| Actinomyces naeslundii MG-1 | Akkermansia muciniphila | Bacteroides caccae|
| Bacteroides vulgatus | Bifidobacterium dentium | Bifidobacterium longum subsp. infantis |
| Corynebacterium accolens | Corynebacterium amycolatum | Corynebacterium matruchotti |
| Enterococcus faecalis | Escherichia coli | Eubacterium brachy |
| Faecalibacterium prausnitzii | Fusobacterium nucleatum subsp. nucleatum | Gardnerella vaginalis |
| Lactobacillus crispatus | Lactobacillus gasseri | Lactobacillus iners |
| Lactobacillus jensenii | Lautropia mirabilis | Moraxella catarrhalis |
| Porphyromonas gingivalis | Prevotella melaninogenica | Prevotella nigrescens| 
| Prevotella oralis | Cutibacterium acnes | Rothia dentocariosa |
| Ruminococcus lactaris | Staphylococcus aureus subsp. aureus | Streptococcus agalactiae |
| Streptococcus mutans | Streptococcus pneumoniae | Streptococcus sanguinis|
| Tannerella forsythia | Treponema denticola | Veillonella parvula |

<br>

#### A (non-exhaustive) list of points to consider:

>How many ASVs were generated by Dada2? How does this compare to the number of unique taxa present
in this mock community?

>What impact does using a denoising approach like Dada2 have on sample richness estimations? 

>What proportion of ASVs could be identified to species level? 

>To what extent does taxonomic classification depend on a) the approach and b) the reference database used?

(NB. you can rerun the above code using alternative reference database, which are available for use with
Dada2 [here][dada2_ref].)

>How many ASVs could be matched to each genus present in the mock community?

>How do these approaches handle intragenomic 16S gene copy variation?

>How does using ASVs compare to using OTUs for community-level microbiome analysis?


[med]: https://www.ncbi.nlm.nih.gov/pmc/articles/PMC4817710/
[unoise]: https://doi.org/10.1101/081257
[dada2]: https://www.ncbi.nlm.nih.gov/pmc/articles/PMC4927377/
[deblur]: https://www.ncbi.nlm.nih.gov/pmc/articles/PMC5340863/
[dnd]: https://www.ncbi.nlm.nih.gov/pmc/articles/PMC6087418/
[perspective]: https://www.ncbi.nlm.nih.gov/pmc/articles/PMC5702726/
[dada2_pipe]: https://benjjneb.github.io/dada2/tutorial.html
[dada2_ref]: https://zenodo.org/record/6655692
[apply]: https://towardsdatascience.com/dealing-with-apply-functions-in-r-ea99d3f49a71
[splappcom]: https://www.jstatsoft.org/article/view/v040i01
[dadadb]: https://benjjneb.github.io/dada2/training.html
[blast]: https://blast.ncbi.nlm.nih.gov/Blast.cgi?PROGRAM=blastn&PAGE_TYPE=BlastSearch&LINK_LOC=blasthome
[max_target_seqs]: https://academic.oup.com/bioinformatics/article/35/9/1613/5106166
[blastcmd]: https://www.ncbi.nlm.nih.gov/books/NBK279671/
[operon_divergence]: https://www.ncbi.nlm.nih.gov/pmc/articles/PMC387781/