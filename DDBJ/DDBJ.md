

DDBJ data management
====================

Raw sequences in FASTQ format were deposited in the DNA DataBank of Japan
(DDBJ), with metadata relating each file to one multiplexed sample from one
HiSeq library.  This page explains how to associate sequencing data to a given
well from a given C1 run, and provides scripts to download the data and rename
the files according to cell identifiers instead of DDBJ accession numbers.  The
first part, on metadata handling is done in `R` and is rebuildable.  The second
part, which involves a long download, has been tested but is not automatically
re-executed when runnign `knitr`.


Accession numbers and metadata
------------------------------

The following script downloads metadata from DDBJ and creates a table that has 6 columns and 472 rows. Each row represents a single cell. The column names are the following:

### `Library` 

the sequencing flowcell run ID (e.g. `RNhi10371`)

### `Well` 

the coordinate of a cell on the 96-well plate after transfer of cDNA from the C1 capture array 

### `DDBJ`

the unique cell ID given to each cell by DDBJ

### `Barcode`

the left and right read (2 x 8 bases) barcode used to pool 96 cells per run

### `ExperimentAccession`

the name given by DDBJ that is the equivalent of the `Library` run ID 

### `Run`

the C1 capture array ID (e.g. `1772-062-248`)

### `cell_id`

a unique identifier for each single cell, that is created by combining the C1 capture array  `Run` ID and the `Well` with an underscore delimeter



```r
library(XML)
```

```
## Warning: package 'XML' was built under R version 3.1.2
```

```
## Loading required package: methods
```

```r
library(reshape)
```

The XML metadata is downloaded from DDBJ, and parsed in R. Informations such
as run ID, DDBJ accession numbers and Barcodes are extracted with XPath
queries.


```r
DDBJ <- xmlTreeParse("ftp://ftp.ddbj.nig.ac.jp/ddbj_database/dra/fastq/DRA002/DRA002399/DRA002399.experiment.xml", useInternal=T)
top        <- xmlRoot(DDBJ)
RunID      <- xpathApply(top, "//LIBRARY_NAME", xmlValue)
Experiment <- xpathApply(top, "//EXPERIMENT", xmlAttrs)
Accession  <- lapply(Experiment, function(x) x[1])
Barcodes   <- xpathApply(top, "//LIBRARY_CONSTRUCTION_PROTOCOL", xmlValue)
Barcodes   <- gsub("\n", "", as.character(Barcodes))
Barcodes   <- strsplit(Barcodes, split=', ')
names(Barcodes) <- paste(RunID, Accession, sep="_")
```

A correspondance table between barcodes and well IDs (from the 96-well plates)
was prepared by hand according to the Fluidigm protocol 'PN 100-5950 A1' and
saved in text format. It is loaded in R with the following command.


```r
WellID <- read.csv("WellToBarcodes.csv", header=T, stringsAsFactors=F)
```

Each Fluidigm run produced one multiplexed library that was sequenced on one
lane, so there is a one-to-one correspondence between run IDs and sequence
library IDs, as described in the vector below.  See the [HiSeq QC page](../HiSeq/HiSeq.md)
for more information.


```r
LibraryToC1ID <- c( RNhi10371="1772-062-248"
                  , RNhi10372="1772-062-249"
                  , RNhi10395="1772-064-103"
                  , RNhi10396="1772-067-038"
                  , RNhi10397="1772-067-039")
```

The metadata is then aggreated with the well IDs in a single table.


```r
# create combined table in long format with run ID and DDBJ ID and barcode
link <- melt(Barcodes)
names(link) <- c("DDBJBarcode", "Library")

# separate DDBJ name and barcode
intermediate <- read.table( text=as.character(link$DDBJBarcode), sep=':'
                          , col.names=c("DDBJ","Barcode"))

# split RunID and ExperimentAccession in column 1 of final
intermediate2 <- read.table( text=as.character(link$Library), sep='_'
                           , col.names=c("Library", "ExperimentAccession"))

# Paste the intermediate tables as a new 'link' table.
link <- cbind(intermediate, intermediate2)

# associate barcode from the DDBJ XML with the Well coordinates
link$Well <- WellID$Well[match(as.character(link$Barcode), as.character(WellID$Barcode))]

# get cell IDs for DDBJ names
link$Run <- LibraryToC1ID[link$Library]
link$cell_id <- paste(link$Run, link$Well, sep="_")
head(link)
```

```
##        DDBJ           Barcode   Library ExperimentAccession Well          Run          cell_id
## 1 DRR028133 AAGAGGCA-AAGGAGTA RNhi10371           DRX019711  G11 1772-062-248 1772-062-248_G11
## 2 DRR028134 AAGAGGCA-ACTGCATA RNhi10371           DRX019711  F11 1772-062-248 1772-062-248_F11
## 3 DRR028135 AAGAGGCA-AGAGTAGA RNhi10371           DRX019711  D11 1772-062-248 1772-062-248_D11
## 4 DRR028136 AAGAGGCA-CTAAGCCT RNhi10371           DRX019711  H11 1772-062-248 1772-062-248_H11
## 5 DRR028137 AAGAGGCA-CTCTCTAT RNhi10371           DRX019711  B11 1772-062-248 1772-062-248_B11
## 6 DRR028138 AAGAGGCA-GTAAGGAG RNhi10371           DRX019711  E11 1772-062-248 1772-062-248_E11
```


```r
# write final output table to current working directory
write.csv(link, "DDBJLink.csv", row.names=F)
```

Data download and file renaming
-------------------------------

The following description is only suitable for the download and renaming of fastq file stored at [DDBJ](http://trace.ddbj.nig.ac.jp/DRASearch/submission?acc=DRA002399). However, the same data can be obtained from [NCBI](http://www.ncbi.nlm.nih.gov/Traces/sra/?study=DRP002435) and [EBI](https://www.ebi.ac.uk/ena/data/view/DRP002435) 

This script, and the page on [spike detection](../control-sequences/control-sequences.md) assume
that the 945 compressed fastq files are downloaded in this directory.  If you
downoload them somewhere else, you need to provide symbolic links, like with
`for FILE in /somewhere/else/*.fastq.bz2 ; do ln -s $FILE ; done`.


```sh
lftp ftp://ftp.ddbj.nig.ac.jp/ddbj_database/dra/fastq/DRA002/DRA002399
mget -c */*.bz2
```

The next part utilises the file `DDBJLink.csv` to rename downloaded fastq.bz2 files with the unique `cell_id`.
Adjust the below script to the directory in which the `DDBJLink.csv` is located. Furthermore, your working directory should be `yourdirectoryname` where all the dowloaded fastq.bz2 files are saved. 


```sh
# the code below replaces the "DDBJ" name of each fastq.bz2 file pair with the corresponding "cell_id"
cut -f4,8 DDBJLink.csv | sed 1d | while read from to ; do mv ${from}_1.fastq.bz2 ${to}.1.fastq.bz2 ; mv ${from}_2.fastq.bz2 ${to}.2.fastq.bz2; done
```

The file fastq_md5sum.csv contains the md5 checksums for all fastq.bz2 files after renaming and can be used for validity checks by comparing the md5sums of your local renamed files.


```sh
md5sum -c fastq_md5sums.txt
```
