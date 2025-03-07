# DATA PRE-PROCESSING
This script performs pre-processing of the raw sequencing data to generate UMI count tables in multiple steps. Script primarily runs in bash or linux shell and makes use of few tools such as Illumin's bcl2fastq, 10X Genomics's CellRanger, FASTQC, STAR, featureCounts, UMI Tools, etc.




#### DEMULTIPLEXING
- demultiplex the raw sequencing data BCL files
- resulting files are fastq.gz files
- uses Illumina's *bcl2fastq* and 10X Genomics *CellRanger* software program

```bash
cellranger mkfastq --run=<path to BCL files> --samplesheet=<path to sample sheet>
```

  

#### COMBINING FASTQ FILES
- demultiplexed fastq files are usually separated by lanes
- *cat* command (bash) can combine multiple lanes into a single fastq file
- *samples.txt* contains list of samples

```bash
cat samples.txt | while read line
  do
    myLibrary=`echo $line`
    echo "Processing" ${myLibrary}
    cat ${myLibrary}/${myLibrary}*R1*.gz > ${myLibrary}_R1.fastq.gz
    cat ${myLibrary}/${myLibrary}*R2*.gz > ${myLibrary}_R2.fastq.gz
  done
```
  



#### QUALITY CHECK
- run quality check on the fastq.gz files
- uses *FASTQC* software program

```bash
ls *.fastq.gz | sed "s/.fastq.gz//g" | xargs -I % -n 1 -P 48 sh -c 'echo %; fastqc %.fastq.gz'
```

  


#### WHITELIST
- estimate and create a *whitelist* of real cell barcodes from the quality filtered fastq.gz files
- uses UMI Tools software program

```bash
ls *R1*.gz | sed "s/_R1.fastq.gz//g" | xargs -I % -n 1 -P 48 sh -c 'echo %; umi_tools whitelist --stdin=%_R1.fastq.gz --bc-pattern=CCCCCCCCCCCCCCCCNNNNNNNNNN --expect-cells=<number of expected cells> --plot-prefix=%_Expect_Whitelist --log=%_Whitelist_log.out --error=%_Whitelist_log.err --stdout=%_Whitelist.txt'
```




#### EXTRACT READS
- *extract* the reads corresponding to estimated whitelist of real cell barcodes
- also appends the cell-barcode and umi information from R1 fastq files to read names of R2 fastq files
- uses UMI Tools software program

```bash
ls *R1*.gz | sed "s/_R1.fastq.gz//g" | xargs -I % -n 1 -P 48 sh -c 'echo %; umi_tools extract --bc-pattern=CCCCCCCCCCCCCCCCNNNNNNNNNN --stdin=%_R1.fastq.gz --stdout=%_R1_extracted.fastq.gz --read2-in=%_R2.fastq.gz --read2-out=%_R2_extracted.fastq.gz --filter-cell-barcode --whitelist=%_Whitelist.txt --log=%_Extract_log.out --error=%_Extract_log.err'
```

  


#### READS ALIGNMENT
- align the extracted reads with reference genome and annotation
- uses *STAR* software program

###### Prior alignment, reference genome index need to be built
- This is one-time only step and can be used multiple times for same genome build
- genome build: mouse genome GRCm38.p6 (MM10)
- annotation: gencode vM17

```bash
STAR --runMode genomeGenerate --runThreadN 48 --genomeDir <genome index directory> --genomeFastaFiles <reference genome fasta  file> --sjdbGTFfile <reference annotation gtf file> --sjdbOverhang 100
```




###### Once the genome index is ready, run the alignment as below

```bash
for FQFILE in `ls *R2_extracted*.gz`
 do
  prefx=`echo ${FQFILE} | sed "s/_R2_extracted.fastq.gz//g"`
  echo "Processing" ${prefx}

  STAR --runThreadN 48 \
       --genomeDir <path the reference genome index> \
       --readFilesIn ${FQFILE} \
       --readFilesCommand zcat \
       --sjdbGTFfile <path to reference annotation gtf file> \
       --outFilterType BySJout  \
       --outFilterMismatchNoverReadLmax 0.04 \
       --outFilterMultimapNmax 10 \
       --alignSJoverhangMin 10 \
       --alignSJDBoverhangMin 1 \
       --outSAMtype BAM SortedByCoordinate \
       --outFilterMismatchNmax 5 \
       --twopassMode Basic \
       --outFileNamePrefix ${prefx}_STAR_
 done
```




#### READS ASSIGNMENT
- assign the aligned reads with reference annotation
- uses *featureCounts* software program

```bash
for BAMFILE in `ls *_STAR_Aligned.sortedByCoord.out.bam`
 do
  prefx=`echo ${BAMFILE} | sed "s/_STAR_Aligned.sortedByCoord.out.bam//g"`
  echo ${prefx}
  
  featureCounts \
    --primary \
    -R BAM \
    -T 48 \
    -s 1 \
    -t gene \
    -g gene_name \
    -a <path to reference annotation gtf file> \
    -o ${prefx}_Primary_Gene_Assigned \
    ${BAMFILE}

 done
```




#### SORT AND INDEX BAM FILE
- *sort* and *index* the assigned bam file
- uses *Samtools* software program

```bash
for BFILE in `ls *featureCounts.bam` 
 do
  outfile=`echo ${BFILE} | sed "s/.sortedByCoord.out.bam.featureCounts.bam/_Assigned_Sorted.bam/g"` 
  samtools sort ${BFILE} -o ${outfile}
  samtools index ${outfile}
 done
```




#### COUNT UMI PER GENE PER CELL
- generate *count* table from assigned and sorted reads
- uses UMI Tools software program

```bash
ls *_STAR_Aligned_Assigned_Sorted.bam | sed "s/_STAR_Aligned_Assigned_Sorted.bam//g" | xargs -I % -n 1 -P 48 sh -c 'echo %; umi_tools count --per-gene --gene-tag=XT --per-cell --stdin=%_STAR_Aligned_Assigned_Sorted.bam --stdout=%_Counts.tsv.gz --log=%_Counts.log --error=%_Counts.err --wide-format-cell-counts'
```

The resulting file is a table with *rows* as genes and *columns* as individual cell-ids with information for UMI counts per gene per cell.



