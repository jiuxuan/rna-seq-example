# Aligned Read Processing and Counting



```r
opts_chunk$set(fig.width = 7, fig.height = 7, cache = TRUE)
opts_knit$set(base.url = "https://github.com/vsbuffalo/rna-seq-example/raw/master/")
```




## Load Required Packages and Set Number of Cores to Use



```r
library(ggplot2)
library(multicore)

library(GenomicRanges)
library(Rsamtools)
library(GenomicFeatures)

library(org.At.tair.db)
library(TxDb.Athaliana.BioMart.plantsmart12)

options(mc.cores = 4)
```





## Build Exons By Gene `GRangesList` Object from `transcriptDb` Object

This analysis will be gene-level; our first step will be to take the
A. thaliana `transcriptDb` object and use the `exonsBy` method to
group exons by gene. We could also specify `by=tx` to do the analysis
at the transcript level. It's not a bad idea to check how many
elements (genes) this creates:



```r
txdb <- TxDb.Athaliana.BioMart.plantsmart12
at.exons <- exonsBy(txdb, "gene")
length(at.exons)
```

```
## [1] 33602
```




The `show` method for `txdb` reveals important data about the database
versions, curation dates, etc, all vital for reproducibility.

### Adjusting sequence levels

The alignment and exon objects present us with a common problem well
worth discussing: they have different sequence naming schemes. 

Here is an illustration of the problem (we read in an alignment file
that we then discard to save memory):



```r

aln <- readBamGappedAlignments("data/alignments/SRR070571-trimmed-final.bam")

seqlevels(aln)
```

```
## [1] "Chr1"         "Chr2"         "Chr3"         "Chr4"        
## [5] "Chr5"         "chloroplast"  "mitochondria"
```

```r

seqlevels(at.exons)
```

```
## [1] "3"  "4"  "1"  "5"  "2"  "Pt" "Mt"
```




These are differently named (even though they correspond to the same
chromosomes), so we must rename them. We could adjust manually with
`seqlevels(at.exons) <- # blah, blah, blah`, but it's preferred that
we use `renameSeqlevels`:



```r

new.names <- c(`1` = "Chr1", `2` = "Chr2", `3` = "Chr3", `4` = "Chr4", 
    `5` = "Chr5", Mt = "mitochondria", Pt = "chloroplast")


at.exons <- renameSeqlevels(at.exons, new.names)
setdiff(seqlevels(aln), seqlevels(at.exons))
```

```
## character(0)
```




This corrects this issue.

## Alignment Statistics

Gathering information on the alignment of reads is absolutely crucial
and a large reason why I do this process in R rather than use
[HTSeq](http://www-huber.embl.de/users/anders/HTSeq/doc/count.html?highlight=rna)
or other tools (it's worth mentioning HTSeq is very good software
though).

On most machines however, we hit memory limits that prevent us from
loading in every sample's alignments fully into memory. The workflow
that works well is to process each read's overlaps with exonic regions
individual and then `cbind` the resulting vectors. While each
alignment region is being processed to count data, we can gather
statistics on the alignment file. For this, we'll need functions that
gather such statistics.

Note that when we look at the width of alignments, our interpretation
depends upon prior beliefs. An alignment wider than the median, 75th
quantile, and largest introns all mean different things. We first form
our thoughts about this process by gathering statistics on Arabidopsis
intron lengths. `mclapply` doesn't work on these objects
(`GRangesList` are not list!), so we have to use the `foreach` and
`doMC` packages.



```r
registerDoMC(cores = 4)
# intron lengths
at.intron.lengths <- foreach(x = at.exons, .packages = "GenomicRanges") %dopar% 
    width(gaps(x, start = min(start(x))))

# we use summary to build a vector of cut points
read.length <- 42
cut.points <- sort(c(0, read.length, summary(unlist(at.intron.lengths)), 
    Inf))
```




Now, we get to gathering statistic and count data.



```r

getAlignmentStatistics <- function(aln) {
    ngap <- table(ngap(aln))
    width <- table(cut(width(aln), cut.points))
    mapq <- table(elementMetadata(aln)$mapq)
    xq <- table(elementMetadata(aln)$XQ)
    list(ngap, mapq, width, xq)
}

bam.files <- list.files("data/alignments/", pattern = ".*\\.bam", 
    full.names = TRUE)
FORCE <- TRUE
aln.results.file <- "data/aln-results.Rdata"
if (!file.exists(aln.results.file) || TRUE) {
    aln.results <- mclapply(bam.files, function(f) {
        aln = readBamGappedAlignments(f, param = ScanBamParam(what = c("mapq", 
            "qname"), tag = c("NH", "XQ")))
        counts <- summarizeOverlaps(at.exons, aln, ignore.strand = TRUE)
        list(counts = counts, info = getAlignmentStatistics(aln))
    })
    save(aln.results, file = aln.results.file)
} else {
    load(aln.results.file)
}
```




`aln.results` is a list of lists: the first level deep corresponds to
samples, the second level information per experiment (the first, a
`summarizedExperiment` object from `summarizeOverlaps` and the second,
another list of data from `getAlignmentStatistics`). We'll need to
seperately pull these elements.



```r

aln.counts <- lapply(aln.results, function(x) assays(x[["counts"]])$counts)

names(aln.counts) <- names(aln.results) <- gsub("([A-Z0-9]+)-.*", 
    "\\1", basename(bam.files))

raw.counts <- do.call(cbind, aln.counts)
colnames(raw.counts) <- names(aln.results)

# process alignment info, adding a column so entries can be identified
# after rbind
info.fields <- c("ngap", "mapq", "width", "xq")
aln.info <- mapply(function(x, varn) {
    tmp <- mapply(function(y, n) {
        ll <- y$info[[x]]
        out <- data.frame(ll, sample = n, var = varn)
        colnames(out)[1:2] <- c("value", "frequency")
        out
    }, aln.results, names(aln.results), SIMPLIFY = FALSE)
    tmp
}, seq_along(info.fields), info.fields, SIMPLIFY = FALSE)
names(aln.info) <- info.fields
```




Note that we added the measure name (width, XQ, etc) and the sample
name for easier plotting. 

Alignment width distribution is important: it can tell us whether our
alignments are wider than we expect (maybe too many longer than the
upper quartile for intron length), or whether there's a mapping width by sample effect (which would confound count data):



```r

ggplot(do.call(rbind, aln.info[["width"]])) + geom_bar(aes(x = value, 
    y = frequency, fill = sample), position = "dodge") + opts(axis.text.x = theme_text(angle = 45))
```

![plot of chunk width-plot](https://github.com/vsbuffalo/rna-seq-example/raw/master/figure/width-plot.png) 


Nothing looks too out of the ordinary here. What about mapping quality?



```r
d.mapq <- local({
    tmp <- do.call(rbind, aln.info[["mapq"]])
    within(tmp, {
        value <- as.integer(as.character(value))
    })
})
p <- ggplot(d.mapq) + geom_bar(aes(x = value, y = frequency, fill = sample), 
    position = "dodge", stat = "identity")
p + scale_x_continuous("mapping quality")
```

![plot of chunk mapping-qual-plot](https://github.com/vsbuffalo/rna-seq-example/raw/master/figure/mapping-qual-plot.png) 


So we see there are a lot of low-quality mapped reads. We can get a
clearer photo by subsetting:



```r
p <- ggplot(subset(d.mapq, value <= 10)) + geom_bar(aes(x = value, 
    y = frequency, fill = sample), position = "dodge", stat = "identity")
p + scale_x_continuous("mapping quality")
```

![plot of chunk mapping-bad-qual-plot](https://github.com/vsbuffalo/rna-seq-example/raw/master/figure/mapping-bad-qual-plot.png) 


Note the number of reads that are non-unique:



```r

d.mapq[d.mapq$value == 0, ]
```

```
##             value frequency    sample  var
## SRR070570.1     0   7268457 SRR070570 mapq
## SRR070571.1     0   5694235 SRR070571 mapq
## SRR070572.1     0   1361640 SRR070572 mapq
## SRR070573.1     0   1257715 SRR070573 mapq
```




And that this differs by sample quite a bit. A read recovery/rescue
option may be useful here. 

Normally, I would do more diagnostics, but `qrqc` (or another package)
will incorporate these soon. For now, I output the raw counts table to
be used in the statistical side of things:



```r
write.table(raw.counts, file = "results/raw-counts.txt", quote = FALSE)
```




### Good Alignments by Sample

How many *good* alignments do we have (mapping quality above or equal
to 30) do we have?



```r

aln.rates <- subset(d.mapq, var == "mapq" & value >= 30)
aln.rates$treatment <- ifelse(aln.rates$sample %in% c("SRR070570", 
    "SRR070571"), "wt", "mutant")

ggplot(aln.rates) + geom_bar(aes(x = sample, y = frequency, fill = treatment), 
    stat = "identity")
```

![plot of chunk mapping-rates](https://github.com/vsbuffalo/rna-seq-example/raw/master/figure/mapping-rates.png) 


