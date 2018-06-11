#RNA-seq: exercise 2
##Mapping and aligning the reads:

##Objectives:

* Learn how to align with HISAT2 and kallisto
* Convert the output from the aligners to the BAM format.
* Explore the alignments

##Loading the programs to use

To load the tools we will use in the tutorial run:

```sh
hisat2/2.0.5
kallisto/0.43.0
samtools/1.8
stringtie/1.0.4
``` 

* As before, Keep the record of which version of the software you use. Sometimes updates may change the output. To be consistent in the analysis, **use always the same version** within a experiment. 
* You can write scripts with all your commands to run the same analysis again.

### Outputs
For this tutorial, create folders for the outputs of each program. 

```sh
mkdir -p alignments
mkdir -p alignments/kallisto
mkdir -p alignments/HISAT2
```

##Run the alignments using the previously trimmed PE reads

###HISAT2
The first step is to run the alignments on each sample individually, and then pipe the output through samtools to convert the SAM file to a BAM file.

```
module load hisat2/2.0.5
module load samtools/1.8

hisat2 -x /hisat_index/transformed_coordinates_fasta_indx \
-p 2 --dta \
--known-splicesite-infile gtf_splice/original_coordinates_splices.txt \
-1 /trim/104B_1P.fastq \
-2 /trim/104B_2P.fastq \
| samtools view -bS - > alignments/HISAT2/104B_P.bam
```

* To run the alignments faster, you can increase the number of threads by changing the flag ```-p 8``` in ```hisat2```

All the bam files need to be sorted for downstream analysis/ 

```sh
module load samtools/1.8

samtools sort alignments/HISAT2/104B_P.bam \
-o alignments/HISAT2/sorted_104B_P.bam
```

Repeat the mapping and subsequent sorting of the output BAM file for each sample.

####Questions: 
* Does alignments/HISAT2/ contain all the BAM files you expect?

Look at the STDERR file:
* What was the overall mapping % for each sample?
* What percentage of reads mapped that as pairs, uniquely?
* What percentage of reads mapped that as pairs, more than once?

###StringTie
We use StringTie to quantify the gene expression in each sample. It takes a ```GFF``` or ```GTF``` annotation file 
as input, along with a ```BAM``` file containing RNA-seq read mappings, sorted by their genomic location.

```sh
module load stringtie/1.0.4

stringtie \
alignments/HISAT2/sorted_104B_P.bam \
-p 8 -G gtf_splice/original_coordinates.gtf -l strg -o 104B_P_string.gtf
```
Run Stringtie for each of the 4 samples.
The output ```.gtf``` file for each sample contains the name of each transcript and its coordinates, along with FPKM values.

Next, merge the transcripts from all the samples using ```StringTie --merge```, to create a single ```.gtf``` that we will use to re-quantify gene expression.
By providing the original ```References/original_coordinates.gtf``` at the same time, StringTie will use this as a guide when reconstructing novel transcripts.

We also need to create a ```.txt``` file containing a list of the ```.gtf``` output files generated by StringTie, that will be used for merging.

```sh
104B_P_string.gtf
105B_P_string.gtf
119B_P_string.gtf
120A_P_string.gtf
```

```sh
module load stringtie/1.0.4

stringtie --merge -p 8 -G gtf_splice/original_coordinates.gtf \
-l mrg \
-o gtf_splice/merge_stringtie_gtf gtf_splice/list_gtf.txt 
```

We use this ```gtf_splice/merge_stringtie_gtf``` to re-estimate transcript abundances and create tables of counts for Ballgown.

```sh
module load stringtie/1.0.4

stringtie -e -B -p 8 -G gtf_splice/merge_stringtie_gtf \
-o ./ballgown_str_mrg/104B/104B_vs_mrg_gtf.gtf \
alignments/HISAT2/sorted_104B_P.bam
```

The option ```-e``` limits the processing of read alignments to only estimate and output the assembled 
transcripts matching the reference transcripts given with the ```-G``` option (requires ```-G```, recommended for ```-B/-b```). 
With this option, read bundles with no reference transcripts will be entirely skipped, which may provide 
a considerable speed boost when the given set of reference transcripts is limited to a set of target genes.

This option ```-B``` enables the output of 'Ballgown' input table files ```*.ctab``` containing coverage data 
for the reference transcripts given with the ```-G``` option.

If the option ```-o``` is given as a full path to the output transcript file, StringTie will write the ```*.ctab``` 
files in the same directory as the output GTF.

The output ```.gtf``` file for each sample contains the name of each gene, its coordinates, along with FPKM values.

Repeat this using each sorted ```BAM``` file.

####Questions:

* Check that you have the correct folders within the ```ballgown_str_mrg``` directory
* Does each sample folder contain a ```*.ctab``` file?
* Use ```less``` to look at the columns in the ```.gtf``` files generated by StringTie


###kallisto
We will now repeat the mapping, but this time we will use kallisto to map reads to the transcriptome.

```sh
module load kallisto/0.43.0

kallisto quant --pseudobam -i kall_index/kall_selected_refseq1 -t 1 -b 10\
-o /alignments_kallisto/kall_104B \
/trim/104B_1P.fastq \
/trim/104B_2P.fastq \
```

```quant``` runs the quantification algorithm, using the index defined at ```-i``` that we created earlier.
Using the ```---pseudobam``` option outputs pseudoalignments to the transcriptome as a BAM file.
In versions of kallisto prior to 0.44.0, this pseudobam is sent to STDOUT, so we will use samtools to sort it and
save it as a ```bam``` file.
In this kallisto version, we can only specify ```-t 1``` if we are using the ```--pseudobam``` option.

```sh
module load samtools/1.8

samtools sort alignments_kallisto/104B.out \
-o /alignments_kallisto/kall_sorted_104B_P.bam
```

Repeat this for the remaining samples. 
The STERR file contains information on the mapping process.

```kallisto quant``` produces three output files by default:

* ```abundances.h5``` is a HDF5 binary file containing run info, abundance estimates, bootstrap estimates, and transcript length information length. This file can be read in by sleuth
* ```abundances.tsv``` is a plaintext file of the abundance estimates. It does not contains bootstrap estimates. Please use the --plaintext mode to output plaintext abundance estimates. The first line contains a header for each column, including estimated counts, TPM, effective length.
* ```run_info.json``` is a json file containing information about the run

####Questions:
* Check that you have all the output files you expect from running kallisto, and sorting the ```bam``` files
* Use ```less``` to look at the ```abundances.tsv``` file

##Useful links
* **HISAT2 manual**:
    https://ccb.jhu.edu/software/hisat2/manual.shtml
* **StringTie manual**:
    http://ccb.jhu.edu/software/stringtie/index.shtml?t=manual
* **kallisto manual**:
    https://pachterlab.github.io/kallisto/manual
* **SAM Format specification**: https://samtools.github.io/hts-specs/SAMv1.pdf
* **SAM Flags calculator**: https://broadinstitute.github.io/picard/explain-flags.html
