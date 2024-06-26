---
layout: default
title: phase_rare
nav_order: 3
parent: Documentation
---
# phase_rare
{: .no_toc .text-center }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

### Description
Tool to phase rare variants onto a scaffold of common variants (output of phase_common + ligate).
We recommend to use phase_rare for datasets with a sample size greater than 2,000 samples.
For smaller smaple sizes, phase_common could do the job.  

### Usage1: phasing unrelated samples

Phasing large sequencing datasets happens in multiple steps. 

First, let's phase common variants (MAF>0.1%) using phase_common.
<div class="code-example" markdown="1">
```bash
phase_common --input wgs/target.unrelated.bcf --filter-maf 0.001 --region 1 --map info/chr1.gmap.gz --output tmp/target.scaffold.bcf --thread 8
```
</div>

Second, let's use the resulting haplotypes as a scaffold onto which rare variants are phased:
<div class="code-example" markdown="1">
```bash
while read LINE; do
	CHK=$(echo $LINE | awk '{ print $1; }')
	SRG=$(echo $LINE | awk '{ print $3; }')
	IRG=$(echo $LINE | awk '{ print $4; }')
	phase_rare --input wgs/target.unrelated.bcf --scaffold tmp/target.scaffold.bcf --map info/chr1.gmap.gz --input-region $IRG --scaffold-region $SRG --output tmp/target.phased.chunk$CHK\.bcf  --thread 8
done < info/chunks.coordinates.txt
```
</div>

All chunk coordinates are given in the file `info/chunks.coordinates.txt`. This file should be generated using the chunking tool of [GLIMPSE](https://github.com/odelaneau/GLIMPSE).
In this toy example, the scaffold chunks are 3.5Mb and the input chunks are 2.5Mb. Only rare variants within the 2.5Mb regions are phased using a scaffold spanning 3.5Mb (0.5Mb buffer is used on each side for rare variants).

Notes:
- The region defined by \-\-scaffold-region must be larger than \-\-input-region.
- The file `tmp/target.scaffold.bcf` can also be generated by the ligate tool in case phase_common was also run in chunks.
  
Finally, we can bring all chunks of data together by concatenating files using bcftools concat --naive:
<div class="code-example" markdown="1">
```bash
ls -1v tmp/target.phased.chunk*.bcf > tmp/files.txt
bcftools concat -n -Ob -o target.phased.bcf -f tmp/files.txt
bcftools index target.phased.bcf
```
</div>

---

### Usage2: phasing related samples
To take into account duo/trio information while phasing, just give a FAM file specifying the family structures to the --pedigree option.
This file contains one line per sample having parent(s) in the dataset and three columns (kidID fatherID and motherID), separated by TABs for spaces.
You must give the exact same file to the three programs: phase_common, ligate and phase_rare.

To phase the WGS example dataset with family information, run:
<div class="code-example" markdown="1">
```bash
phase_common --input wgs/target.family.bcf --filter-maf 0.001 --pedigree info/target.family.fam --region 1 --map info/chr1.gmap.gz --output tmp/target.scaffold.bcf --thread 8

while read LINE; do
	CHK=$(echo $LINE | awk '{ print $1; }')
	SRG=$(echo $LINE | awk '{ print $3; }')
	IRG=$(echo $LINE | awk '{ print $4; }')
	phase_rare --input wgs/target.family.bcf --scaffold tmp/target.scaffold.bcf --pedigree info/target.family.fam --map info/chr1.gmap.gz --input-region $IRG --scaffold-region $SRG --output $OUT  --thread 8
done < info/chunks.coordinates.txt

for CHK in $(seq 0 3); do echo tmp/target.phased.chunk$CHK\.bcf >> tmp/chunks.files.txt; done

bcftools concat -n -Ob -o target.phased.bcf -f tmp/chunks.files.txt
bcftools index target.phased.bcf
```
</div>

---

### Usage3: phasing chromosome X data

To phase chromosome X data assuming haploidy for males, run;

<div class="code-example" markdown="1">
```bash
phase_common --input wgs/target.haploid.bcf --filter-maf 0.001 --haploids info/target.haploid.txt --region 1 --map info/chr1.gmap.gz --output tmp/target.scaffold.bcf --thread 8

while read LINE; do
	CHK=$(echo $LINE | awk '{ print $1; }')
	SRG=$(echo $LINE | awk '{ print $3; }')
	IRG=$(echo $LINE | awk '{ print $4; }')
	phase_rare --input wgs/target.haploid.bcf --scaffold tmp/target.scaffold.bcf --haploids info/target.haploid.txt --map info/chr1.gmap.gz --input-region $IRG --scaffold-region $SRG --output $OUT  --thread 8
done < info/chunks.coordinates.txt

for CHK in $(seq 0 3); do echo tmp/target.phased.chunk$CHK\.bcf >> tmp/chunks.files.txt; done

bcftools concat -n -Ob -o target.phased.bcf -f tmp/chunks.files.txt
bcftools index target.phased.bcf
```
</div>

The file `info/target.haploid.txt` contains the list of all the haploid samples (i.e. males).

---
 
### Command line options

#### Basic options

| Option name 	       | Argument| Default  | Description |
|:---------------------|:--------|:---------|:-------------------------------------|
| \-\-help             | NA      | NA       | Produces help message |
| \-\-seed             | INT     | 15052011 | Seed of the random number generator  |
| \-T \[ \-\-thread \] | INT     | 1        | Number of thread used|

#### Input files

| Option name 	       | Argument| Default  | Description |
|:---------------------|:--------|:---------|:-------------------------------------|
| \-\-input  		   | STRING  | NA       | Genotypes to be phased in plain VCF/BCF format |
| \-\-input-region     | STRING  | NA       | Region to be considered in \-\-input-plain |
| \-\-scaffold         | STRING  | NA       | Scaffold of haplotypes in VCF/BCF format  |
| \-\-scaffold-region  | STRING  | NA       | Region to be considered in \-\-scaffold  |
| \-\-map              | STRING  | NA       | Genetic map  |
| \-\-pedigree         | STRING  | NA       | Pedigree information (offspring father mother) |
| \-\-haploids         | STRING  | NA       | List of samples that are haploid (e.g. males on chrX) |


#### PBWT parameters

| Option name 	      | Argument|  Default  | Description |
|:--------------------|:--------|:----------|:-------------------------------------|
| \-\-pbwt-modulo     | FLOAT   | 0.1       | Storage frequency of PBWT indexes in cM |
| \-\-pbwt-depth-common | INT     | 2         | Depth of PBWT indexes at common sites to condition on  |
| \-\-pbwt-depth-rare | INT     | 2         | Depth of PBWT indexes at rare sites to condition on  |
| \-\-pbwt-mac        | INT     | 2         | Minimal Minor Allele Count at which PBWT is evaluated |
| \-\-pbwt-mdr        | FLOAT   | 0.1       | Maximal Missing Data Rate at which PBWT is evaluated |

#### HMM parameters

| Option name 	      | Argument|  Default  | Description |
|:--------------------|:--------|:----------|:-------------------------------------|
| \-\-effective-size  | INT     | 15000     | Effective size of the population |

#### Output files

| Option name 	       | Argument| Default  | Description |
|:---------------------|:--------|:---------|:-------------------------------------|
| \-O \[\-\-output \]  | STRING  | NA       | Phased haplotypes in VCF/BCF format |
| \-\-output-buffer    | STRING  | NA       | If specified, right and left buffers are printed in output |
| \-\-log              | STRING  | NA       | Log file  |
