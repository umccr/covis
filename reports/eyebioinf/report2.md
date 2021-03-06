---
title: "Coverage Plotting from Scratch"
author: "Peter Diakumis"
date: "Thu 2018-Mar-08"
output: 
  html_document: 
    keep_md: yes
---

<!-- vim-markdown-toc GFM -->

* [Introduction](#introduction)
* [Requirements](#requirements)
    * [Software](#software)
    * [Datasets](#datasets)
        * [mosdepth BED file](#mosdepth-bed-file)
        * [UCSC Exon BED file](#ucsc-exon-bed-file)
        * [GENCODE TranscriptID to HGNC map file](#gencode-transcriptid-to-hgnc-map-file)
        * [GENCODE GTF file](#gencode-gtf-file)
* [Processing](#processing)
    * [Convert UCSC BED files from hg19 to Ensembl b37](#convert-ucsc-bed-files-from-hg19-to-ensembl-b37)
    * [Intersect UCSC BED files with mosdepth BED file](#intersect-ucsc-bed-files-with-mosdepth-bed-file)
* [Analysis](#analysis)
* [Plot Draft 1](#plot-draft-1)
* [Plot Draft 2](#plot-draft-2)

<!-- vim-markdown-toc -->


## Introduction

Here I'll try to find a way to plot read depth of coverage over gene
exons, specifically targeting a panel of cancer genes.
Most of this work is based on Dave McGaughy's series of blog posts at his
[eyebioinformatics blog](http://davemcg.github.io/post/let-s-plot-3-base-pair-resolution-ngs-exome-coverage-plots/).


## Requirements

### Software
* Create and activate conda environment with dependencies:

```
# list dependencies and channels in scripts/environment.yml:
name: covis
channels:
    - bioconda
dependencies:
    - mosdepth
    - bedtools

# create 'covis' environment based on above file:
conda env create -f scripts/environment.yml

# activate 'covis' environment
source activate covis
```

### Datasets
The following datasets will be required for this report:

1. BED file from mosdepth with per base depth of coverage 
2. BED file from UCSC containing exon/transcript coordinates
3. Text file from GENCODE mapping transcript IDs to HGNC gene names
4. GTF file from GENCODE containing details about gene transcripts
   (what is the difference with 2?)

#### mosdepth BED file

* Can be generated with:

```bash
sample="sample1"
bam="sample1.bam"

mosdepth $sample $bam
```

* Info and content:

```
# wc -l
79,781,011

# head -n6: chr, start, end, depth
1     0 10003 0
1 10003 10052 2
1 10052 10069 3
1 10069 10072 4
1 10072 10075 3
1 10075 10109 4
```


#### UCSC Exon BED file

* Generated with:

There are two possibilities of BED files. The blog author in one part
states that he used the 'Coding Exons' final option, but in the comment section
he states that he used the 'Exons plus 0 bases at each end' final option. His
processing code doesn't work with the second option.
I will therefore generate files using both options and see the differences.

UCSC Table options used:

- clade: Mammal
- genome: Human
- assembly: Feb. 2009 (GRCh37/hg19)
- group: Genes and Gene Predictions
- track: GENCODE Gene V27lift37
- table: Basic (wgEncodeGencodeBasicV27lift37)
- region: genome
- output format: BED - browser extensible data
- output file (**two options**): 
    1. `gencode_genes_v27lift37_exons_padded.bed`
    2. `gencode_genes_v27lift37_exons_coding.bed`
- file type returned: gzip compressed
- get output (**two options**):
    1. Exons plus 0 bases at each end
    2. Coding exons

* Info and content:

```
# wc -l:
gencode_genes_v27lift37_exons_coding.bed.gz
540,381
gencode_genes_v27lift37_exons_padded.bed.gz
651,821


# head -n3: chr,start,end,transcriptID,score,strand
# transcriptID string: transID.versionNumber_X_cds_exonNumber_score_chr_start_f 
./gencode_genes_v27lift37_exons_coding.bed.gz
chr1	67000041	67000051	ENST00000237247.10_1_cds_1_0_chr1_67000042_f	0	+
chr1	67091529	67091593	ENST00000237247.10_1_cds_2_0_chr1_67091530_f	0	+
chr1	67098752	67098777	ENST00000237247.10_1_cds_3_0_chr1_67098753_f	0	+
./gencode_genes_v27lift37_exons_padded.bed.gz
chr1	66999065	66999090	ENST00000237247.10_1_exon_0_0_chr1_66999066_f	0	+
chr1	66999928	67000051	ENST00000237247.10_1_exon_1_0_chr1_66999929_f	0	+
chr1	67091529	67091593	ENST00000237247.10_1_exon_2_0_chr1_67091530_f	0	+
```


#### GENCODE TranscriptID to HGNC map file

* Generated from the [GENCODE website](https://www.gencodegenes.org/)
  after following steps:
    - Data `->`
    - Human `->`
    - Current Release `->`
    - Go to GRCh37 version of this release `->`
    - Metadata files `->`
    - Gene Symbol

* Link [here](ftp://ftp.ebi.ac.uk/pub/databases/gencode/Gencode_human/release_27/GRCh37_mapping/gencode.v27lift37.metadata.HGNC.gz)

* Info and Content:

```
# wc -l
172,844

# head -n5: TranscriptID, HGNC Gene Name
ENST00000456328.2   DDX11L1
ENST00000450305.2   DDX11L1
ENST00000488147.1   WASH7P
ENST00000473358.1   MIR1302-2HG
ENST00000469289.1   MIR1302-2HG
```

#### GENCODE GTF file

* Generated from the [GENCODE website](https://www.gencodegenes.org/)
  after following steps:
    - Data `->`
    - Human `->`
    - Current Release `->`
    - Go to GRCh37 version of this release `->`
    - GTF/GFF3 files `->`
    - Basic gene annotation `->`
    - GTF

* Link [here](ftp://ftp.ebi.ac.uk/pub/databases/gencode/Gencode_human/release_27/GRCh37_mapping/gencode.v27lift37.basic.annotation.gtf.gz)

* Info and Content:

```
# wc -l
1,651,708

# head: chr, source (HAVANA/ENSEMBL), feature type, start, end, score (unused), strand, phase (for CDS features), attributes
##description: evidence-based annotation of the human genome, version 27 (Ensembl 90), mapped to GRCh37 with gencode-backmap - basic transcripts
##provider: GENCODE
##contact: gencode-help@sanger.ac.uk
##format: gtf
##date: 2017-08-01
chr1	HAVANA	gene	11869	14409	.	+	.	gene_id "ENSG00000223972.5_2"; gene_type "transcribed_unprocessed_pseudogene"; gene_name "DDX11L1"; level 2; havana_gene "OTTHUMG00000000961.2_2"; remap_status "full_contig"; remap_num_mappings 1; remap_target_status "overlap";
chr1	HAVANA	transcript	11869	14409	.	+	.	gene_id "ENSG00000223972.5_2"; transcript_id "ENST00000456328.2_1"; gene_type "transcribed_unprocessed_pseudogene"; gene_name "DDX11L1"; transcript_type "processed_transcript"; transcript_name "DDX11L1-202"; level 2; transcript_support_level 1; tag "basic"; havana_gene "OTTHUMG00000000961.2_2"; havana_transcript "OTTHUMT00000362751.1_1"; remap_num_mappings 1; remap_status "full_contig"; remap_target_status "overlap";
chr1	HAVANA	exon	11869	12227	.	+	.	gene_id "ENSG00000223972.5_2"; transcript_id "ENST00000456328.2_1"; gene_type "transcribed_unprocessed_pseudogene"; gene_name "DDX11L1"; transcript_type "processed_transcript"; transcript_name "DDX11L1-202"; exon_number 1; exon_id "ENSE00002234944.1_1"; level 2; transcript_support_level 1; tag "basic"; havana_gene "OTTHUMG00000000961.2_2"; havana_transcript "OTTHUMT00000362751.1_1"; remap_original_location "chr1:+:11869-12227"; remap_status "full_contig";
chr1	HAVANA	exon	12613	12721	.	+	.	gene_id "ENSG00000223972.5_2"; transcript_id "ENST00000456328.2_1"; gene_type "transcribed_unprocessed_pseudogene"; gene_name "DDX11L1"; transcript_type "processed_transcript"; transcript_name "DDX11L1-202"; exon_number 2; exon_id "ENSE00003582793.1_1"; level 2; transcript_support_level 1; tag "basic"; havana_gene "OTTHUMG00000000961.2_2"; havana_transcript "OTTHUMT00000362751.1_1"; remap_original_location "chr1:+:12613-12721"; remap_status "full_contig";
chr1	HAVANA	exon	13221	14409	.	+	.	gene_id "ENSG00000223972.5_2"; transcript_id "ENST00000456328.2_1"; gene_type "transcribed_unprocessed_pseudogene"; gene_name "DDX11L1"; transcript_type "processed_transcript"; transcript_name "DDX11L1-202"; exon_number 3; exon_id "ENSE00002312635.1_1"; level 2; transcript_support_level 1; tag "basic"; havana_gene "OTTHUMG00000000961.2_2"; havana_transcript "OTTHUMT00000362751.1_1"; remap_original_location "chr1:+:13221-14409"; remap_status "full_contig";
```




## Processing

### Convert UCSC BED files from hg19 to Ensembl b37

```bash
gunzip -c gencode_genes_v27lift37_exons_padded.bed.gz | \
# gunzip -c gencode_genes_v27lift37_exons_coding.bed.gz | \
  sed -e 's/^chr//' | \
  sed -e 's/^M/MT/' | \
  grep -v '_gl' | \
  sort -k1,1V -k2,2n | \
  gzip > gencode_genes_v27lift37_exons_padded_b37.bed.gz
# gzip > gencode_genes_v27lift37_exons_coding_b37.bed.gz
```

### Intersect UCSC BED files with mosdepth BED file

```bash
bedtools intersect -wa -wb \
  -a <sample>.per-base.bed.gz \
  -b gencode_genes_v27lift37_exons_coding_b37.bed.gz |\
# -b gencode_genes_v27lift37_exons_padded_b37.bed.gz |\
  bgzip > <sample>.per-base_exons_coding.bed.gz &
# bgzip > <sample>.per-base_exons_padded.bed.gz &
```

* Info and Contents:

```
# wc -l
HCC2218_tumor.per-base_exons_coding.bed.gz
42132787
HCC2218_tumor.per-base_exons_padded.bed.gz
49536854

# head
zcat HCC2218_tumor.per-base_exons_coding.bed.gz | head -n2
1       69090   69091   224     1       69090   70008   ENST00000335137.4_2_cds_0_0_chr1_69091_f        0       +
1       69090   69091   224     1       69090   70008   ENST00000641515.1_1_cds_2_0_chr1_69091_f        0       +
zcat HCC2218_tumor.per-base_exons_padded.bed.gz | head -n2
1       29494   29593   2       1       29553   30039   ENST00000473358.1_1_exon_0_0_chr1_29554_f       0       +
1       29593   29594   1       1       29553   30039   ENST00000473358.1_1_exon_0_0_chr1_29554_f       0       +
```

## Analysis

* Read in metadata and gencode:


```r
metadata <- readr::read_tsv(
  '../../data/ref/gencode.v27lift37.metadata.HGNC.gz',
  col_names = c("transcript", "gene_name"),
  col_types = "cc")

gencode <- readr::read_tsv(
  '../../data/ref/gencode.v27lift37.basic.annotation.gtf.gz',
  col_names = c("chrom", "source", "feature", "start", "end", "score", "strand", "phase", "attribute"),
  col_types = "ccciicccc",
  skip = 5)
```

* Glimpse at datasets:


```r
glimpse(metadata) # 172,844 rows
## Observations: 172,844
## Variables: 2
## $ transcript <chr> "ENST00000456328.2", "ENST00000450305.2", "ENST0000...
## $ gene_name  <chr> "DDX11L1", "DDX11L1", "WASH7P", "MIR1302-2HG", "MIR...
length(table(metadata$gene_name)) # 36,901 unique genes
## [1] 36901
glimpse(gencode) # 1,651,703 rows
## Observations: 1,651,703
## Variables: 9
## $ chrom     <chr> "chr1", "chr1", "chr1", "chr1", "chr1", "chr1", "chr...
## $ source    <chr> "HAVANA", "HAVANA", "HAVANA", "HAVANA", "HAVANA", "H...
## $ feature   <chr> "gene", "transcript", "exon", "exon", "exon", "trans...
## $ start     <int> 11869, 11869, 11869, 12613, 13221, 12010, 12010, 121...
## $ end       <int> 14409, 14409, 12227, 12721, 14409, 13670, 12057, 122...
## $ score     <chr> ".", ".", ".", ".", ".", ".", ".", ".", ".", ".", "....
## $ strand    <chr> "+", "+", "+", "+", "+", "+", "+", "+", "+", "+", "+...
## $ phase     <chr> ".", ".", ".", ".", ".", ".", ".", ".", ".", ".", "....
## $ attribute <chr> "gene_id \"ENSG00000223972.5_2\"; gene_type \"transc...
table(gencode$feature, useNA = "ifany") # 101,235 transcripts
## 
##            CDS           exon           gene Selenocysteine    start_codon 
##         540415         685873          60461             97          55754 
##     stop_codon     transcript            UTR 
##          55704         101235         152164
```

* Filter and extract elements from gencode:


```r
gencode <- gencode %>% 
  filter(feature == "transcript",
         grepl("appris_principal", attribute)) %>% 
  mutate(transcript = str_extract(attribute, 'ENST\\d{11}\\.\\d+'),
         gene_name = str_extract(attribute, 'gene_name\\s\\".*?;'),
         size = end - start) %>%
  separate(gene_name, c('skip', 'gene_name', 'skip2'), '\"') %>% 
  select(-skip, -skip2) %>%
  group_by(gene_name) %>% 
  top_n(n = 1, wt = size) # top gene by size

glimpse(gencode) # down to 20,665 rows
## Observations: 20,665
## Variables: 12
## $ chrom      <chr> "chr1", "chr1", "chr1", "chr1", "chr1", "chr1", "ch...
## $ source     <chr> "HAVANA", "ENSEMBL", "HAVANA", "HAVANA", "ENSEMBL",...
## $ feature    <chr> "transcript", "transcript", "transcript", "transcri...
## $ start      <int> 65419, 134901, 367640, 621059, 738532, 818043, 8611...
## $ end        <int> 71585, 139379, 368634, 622053, 739137, 819983, 8799...
## $ score      <chr> ".", ".", ".", ".", ".", ".", ".", ".", ".", ".", "...
## $ strand     <chr> "+", "-", "+", "-", "-", "+", "+", "-", "-", "+", "...
## $ phase      <chr> ".", ".", ".", ".", ".", ".", ".", ".", ".", ".", "...
## $ attribute  <chr> "gene_id \"ENSG00000186092.5_3\"; transcript_id \"E...
## $ transcript <chr> "ENST00000641515.1", "ENST00000423372.3", "ENST0000...
## $ gene_name  <chr> "OR4F5", "AL627309.1", "OR4F29", "OR4F16", "AL66983...
## $ size       <int> 6166, 4478, 994, 994, 605, 1940, 18837, 5181, 15086...
```

* Read and process coverage data:


```r
# going with 'exons_coding' for now

depth_data <- readr::read_tsv(
  "../../data/HCC2218/mosdepth/HCC2218_tumor.per-base_exons_coding.bed.gz",
  col_names = c("chromA", "startA", "endA", "depth",
                "chromB", "startB", "endB", "transcript", "score", "strand"),
  col_types = "ciiiciiccc")
```


```r
dd<- depth_data %>% 
  separate(transcript, c('transcript', 'rest'), sep = '_cds_') %>% 
  separate(rest, c('before', 'stuff'), sep = '_chr') %>% 
  separate(before, c('exon_number', 'num2'), sep = '_')

dd <- dd %>%
  mutate(
    depth_cat = case_when(
      depth < 10 ~ '< 10 Reads', 
      depth < 20 ~ '< 20 Reads', 
      TRUE ~ '>= 20 Reads')) %>% 
  mutate(
    depth_cat = factor(depth_cat, levels = c('< 10 Reads', '< 20 Reads', '>= 20 Reads'))) %>% 
  mutate(
    transcript = case_when(
      grepl('_', transcript) ~ gsub('_.', '', transcript), 
      TRUE ~ transcript)) %>% 
  select(
    chr = "chromA", start = "startA", end = "endA",
    depth, transcript, strand, depth_cat, exon_number,
    exon_start = "startB", exon_end = "endB")

dd <- dplyr::left_join(dd, metadata, by = c('transcript'))
# saveRDS(dd, "../../data/HCC2218/mosdepth/depth_data2.rds")
```


```r
dd <- readr::read_rds("../../data/HCC2218/mosdepth/depth_data2.rds")
glimpse(dd)
## Observations: 42,132,787
## Variables: 11
## $ chr         <chr> "1", "1", "1", "1", "1", "1", "1", "1", "1", "1", ...
## $ start       <int> 69090, 69090, 69091, 69091, 69092, 69092, 69093, 6...
## $ end         <int> 69091, 69091, 69092, 69092, 69093, 69093, 69094, 6...
## $ depth       <int> 224, 224, 225, 225, 226, 226, 224, 224, 216, 216, ...
## $ transcript  <chr> "ENST00000335137.4", "ENST00000641515.1", "ENST000...
## $ strand      <chr> "+", "+", "+", "+", "+", "+", "+", "+", "+", "+", ...
## $ depth_cat   <fct> >= 20 Reads, >= 20 Reads, >= 20 Reads, >= 20 Reads...
## $ exon_number <chr> "0", "2", "0", "2", "0", "2", "0", "2", "0", "2", ...
## $ exon_start  <int> 69090, 69090, 69090, 69090, 69090, 69090, 69090, 6...
## $ exon_end    <int> 70008, 70008, 70008, 70008, 70008, 70008, 70008, 7...
## $ gene_name   <chr> "OR4F5", "OR4F5", "OR4F5", "OR4F5", "OR4F5", "OR4F...
```


```r
genes <- c('PAX6', 'ABCA4', 'NRL', 'CRX', 'RPGR')
transcripts <- gencode %>%
  filter(gene_name %in% genes) %>%
  pull(transcript)
cbind(genes, transcripts)
##      genes   transcripts         
## [1,] "PAX6"  "ENST00000370225.3" 
## [2,] "ABCA4" "ENST00000379129.7" 
## [3,] "NRL"   "ENST00000561028.5" 
## [4,] "CRX"   "ENST00000221996.11"
## [5,] "RPGR"  "ENST00000318842.11"
```

## Plot Draft 1


```r
# set a custom color that will work even if a category is missing
scale_colour_custom <- function(...){
  ggplot2:::manual_scale('colour', 
                         values = setNames(c('darkred', 'red', 'darkgreen'),
                                           c('< 10 Reads','< 20 Reads','>= 20 Reads')), 
                         ...)
}



plot_maker <- function(tx) {
  num_of_exons <- dd %>%
    filter(transcript == tx) %>%
    pull(exon_number) %>%
    as.numeric() %>% max()
  gene_name <- dd %>%
    filter(transcript == tx) %>% 
    pull(gene_name) %>% 
    unique()
  # expand to create a row for each sequence and fill in previous values
  dd %>%
    filter(transcript == tx) %>% 
    group_by(exon_number) %>% 
    expand(start = full_seq(c(start, end), 1)) %>% 
    # create one row per base position, grouped by Exon Number https://stackoverflow.com/questions/42866119/fill-missing-values-in-data-frame-using-dplyr-complete-within-groups
    # fill missing values https://stackoverflow.com/questions/40040834/r-replace-na-with-previous-or-next-value-by-group-using-dplyr
    left_join(., 
              dd %>%  filter(transcript == tx)) %>% 
    fill(chr:gene_name) %>% 
    ungroup() %>% # drop the exon number grouping, so I can mutate below
    mutate(exon_number = factor(exon_number, levels = 0:num_of_exons)) %>% # Ah, reordering. I need it to be a factor, but then I have to explicitly give the order   
    # mutate(depth_cat = factor(depth_cat, levels = c('< 10 Reads','< 20 Reads','>= 20 Reads'))) %>%  # for coloring
    ggplot(aes(x = start, xend = end, y = depth, yend = depth, colour = depth_cat)) + 
    facet_wrap(~exon_number, scales = 'free_x', nrow = 1, strip.position = 'bottom') + 
    geom_point(size = 0.1) + 
    theme_minimal() + 
    scale_colour_custom() +  # use my custom color set above for my three categories
    theme(axis.text.x = element_blank(), 
          axis.ticks.x = element_blank(), 
          panel.grid.minor = element_blank(), 
          panel.grid.major.x = element_blank(),
          legend.position = 'none') + 
    ylab('Depth') + 
    xlab(paste0(gene_name[1]))
}
```

```r
plots <- purrr::map(transcripts, plot_maker)
## Joining, by = c("exon_number", "start")
## Joining, by = c("exon_number", "start")
## Joining, by = c("exon_number", "start")
## Joining, by = c("exon_number", "start")
## Joining, by = c("exon_number", "start")
cowplot::plot_grid(plotlist = plots, ncol = 1)
```

![](report2_files/figure-html/plot_all2-1.png)<!-- -->

## Plot Draft 2


```r
dd2 <- dd %>%
  filter(transcript %in% transcripts) %>%
  group_by(transcript, exon_number) %>% 
  expand(start = full_seq(c(start, end), 1)) %>% 
  # create one row per base position, grouped by Exon Number https://stackoverflow.com/questions/42866119/fill-missing-values-in-data-frame-using-dplyr-complete-within-groups
  left_join(.,  dd %>% filter(transcript %in% transcripts)) %>% 
  # fill missing values https://stackoverflow.com/questions/40040834/r-replace-na-with-previous-or-next-value-by-group-using-dplyr
  fill(chr:gene_name) 
## Joining, by = c("transcript", "exon_number", "start")
```


```r
dd2 <- dd2 %>%
  group_by(gene_name) %>%
  mutate(pos = 1:n())
even_odds_marking <- dd2 %>%
  group_by(gene_name, exon_number) %>%
  summarise(start = min(pos), end = max(pos)) %>%
  mutate(
    exon = case_when(
      as.numeric(exon_number) %% 2 == 0 ~ 'even',
      TRUE ~ 'odd'))


plot_data <- bind_rows(dd2, even_odds_marking)
```


```r
ggplot() + 
  geom_point(data = plot_data %>% filter(is.na(exon)),
             aes(x = pos, y = depth, colour = depth_cat), size = 0.1) + 
  facet_wrap(~gene_name, ncol = 1) + 
  geom_rect(data = plot_data %>% filter(!is.na(exon)), 
            aes(xmin = start, xmax = end, ymin = -Inf, ymax = Inf, fill = exon)) +  
  scale_fill_manual(values = alpha(c("gray", "white"), .3)) +  
  scale_colour_custom() +
  theme_minimal() +  
  theme(axis.text.x = element_blank(), 
        axis.ticks.x = element_blank(),
        panel.grid.minor = element_blank(), 
        panel.grid.major.x = element_blank(),
        legend.position = "none") +
  ylab('Read Depth') 
```

![](report2_files/figure-html/plot_all3-1.png)<!-- -->

