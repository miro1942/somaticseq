# SomaticSeq

* SomaticSeq is an ensemble caller that has the ability to use machine learning to filter out false positives. The detailed documentation is included in the package, located in [docs/Manual.pdf](docs/Manual.pdf "User Manual"). A quick guide can also be found [here](http://bioinform.github.io/somaticseq/).
* SomaticSeq's open-access paper: [Fang LT, Afshar PT, Chhibber A, et al. An ensemble approach to accurately detect somatic mutations using SomaticSeq. Genome Biol. 2015;16:197](http://dx.doi.org/10.1186/s13059-015-0758-2 "Fang LT, Afshar PT, Chhibber A, et al. An ensemble approach to accurately detect somatic mutations using SomaticSeq. Genome Biol. 2015;16:197.").
* Feel free to report issues and/or ask questions at the [Issues](../../issues "Issues") page. You may also email Li Tai Fang at [li_tai.fang@roche.com](li_tai.fang@roche.com).
* The [v2 branch](../../tree/v2) will continue to be supported in the foreseeable future for bug fixes, but new features will only be introduced in the current master branch. 

## Requirements
* Python 3, plus pysam, numpy, and scipy libraries.
* R, plus [ada](https://cran.r-project.org/package=ada) library: required in training or prediction mode
* [BEDTools](https://bedtools.readthedocs.io/en/latest/): required when parallel processing in invoked, and/or when any bed files are used as input files
* Optional: dbSNP VCF file (if you want to use dbSNP membership as a feature).
* At least one of the callers we have incorporated, i.e., MuTect2 (GATK4) / MuTect / Indelocator, VarScan2, JointSNVMix2, SomaticSniper, VarDict, MuSE, LoFreq, Scalpel, Strelka2, TNscope, and/or Platypus.

## Example commands
* At minimum, given the results of the individual mutation caller(s), SomaticSeq will extract sequencing features for the combined call set. Required inputs are `--output-directory`, `--genome-reference`, `paired|single`, `--tumor-bam-file`, and `--normal-bam-file`. Everything else is optional.
* The following four files will be created into the output directory:
  * *Consensus.sSNV.vcf*, *Consensus.sINDEL.vcf*, *Ensemble.sSNV.tsv*, and *Ensemble.sINDEL.tsv*.

* If you're searching for pipelines to run those individual somatic mutation callers, feel free to take advantage of our [dockerized somatic mutation scripts](utilities/dockered_pipelines).

```
# Merge caller results and extract SomaticSeq features
$somaticseq/somaticseq/run_somaticseq.py \
--output-directory  $OUTPUT_DIR \
--genome-reference  GRCh38.fa \
--inclusion-region  genome.bed \
--exclusion-region  blacklist.bed \
paired \
--tumor-bam-file    tumor.bam \
--normal-bam-file   matched_normal.bam \
--mutect2-vcf       MuTect2/variants.vcf \
--varscan-snv       VarScan2/variants.snp.vcf \
--varscan-indel     VarScan2/variants.indel.vcf \
--jsm-vcf           JointSNVMix2/variants.snp.vcf \
--somaticsniper-vcf SomaticSniper/variants.snp.vcf \
--vardict-vcf       VarDict/variants.vcf \
--muse-vcf          MuSE/variants.snp.vcf \
--lofreq-snv        LoFreq/variants.snp.vcf \
--lofreq-indel      LoFreq/variants.indel.vcf \
--scalpel-vcf       Scalpel/variants.indel.vcf \
--strelka-snv       Strelka/variants.snv.vcf \
--strelka-indel     Strelka/variants.indel.vcf
```

* `--inclusion-region` or `--exclusion-region` will require BEDTools in your path.
* The command can be parallelized using `$somaticseq/somaticseq_parallel.py` script, with identical input options except for `--threads X` placed before the `paired` option to indicate X threads. The `somaticseq_parallel.py` is simply a script to create multiple sub-BED files, and then invokes `somaticseq/run_somaticseq.py` on each of those sub-BED files in parallel, then merge the results. It requires BEDTools in your path.
* For all input VCF files, either .vcf or .vcf.gz are acceptable.

Additional parameters to be specified **before** `paired` option to invoke training mode. In addition to the four files specified above, two additional files (classifiers) will be created, i.e., *Ensemble.sSNV.tsv.ntChange.Classifier.RData* and *Ensemble.sINDEL.tsv.ntChange.Classifier.RData*.

* `--somaticseq-train`: FLAG to invoke training mode with no argument, which also requires the following inputs, R and ada package in R.
* `--truth-snv`:        if you have ground truth VCF file for SNV
* `--truth-indel`:      if you have a ground truth VCF file for INDEL

Additional input files to be specified **before** `paired` option invoke prediction mode (to use classifiers to score variants). Four additional files will be created, i.e., *Seq.Classified.sSNV.vcf*, *SSeq.Classified.sSNV.tsv*,  *SSeq.Classified.sINDEL.vcf*, and *SSeq.Classified.sINDEL.tsv*.
* `--classifier-snv`:   classifier (.RData file) previously built for SNV
* `--classifier-indel`: classifier (.RData file) previously built for INDEL

Without those paramters above to invoking training or prediction mode, SomaticSeq will default to majority-vote consensus mode.


Do not worry if Python throws the following warning. This occurs when SciPy attempts a statistical test with empty data, e.g., z-scores between reference- and variant-supporting reads will be NaN if there is no reference read at a position.

```
  RuntimeWarning: invalid value encountered in double_scalars
  z = (s - expected) / np.sqrt(n1*n2*(n1+n2+1)/12.0)
```

## Dockerized workflows and pipelines

### To run somatic mutation callers and then SomaticSeq
We have created scripts that run all the dockerized somatic mutation callers and then SomaticSeq at [**utilities/dockered_pipelines**](utilities/dockered_pipelines).
All you need is [docker](https://www.docker.com/).

### To create training data to create SomaticSeq classifiers
We have also dockerized pipelines for *in silico* mutation spike in at [**utilities/dockered_pipelines/bamSimulator**](utilities/dockered_pipelines/bamSimulator).
These pipelines are based on [BAMSurgeon](https://github.com/adamewing/bamsurgeon). We use it to create training set to build SomaticSeq classifiers.

### GATK's best practices for alignment
The limited pipeline to generate BAM files based on GATK's best practices is at [utilities/dockered_pipelines/alignments](utilities/dockered_pipelines/alignments).

### Additional workflows
* A [Snakemake](https://snakemake.readthedocs.io/en/latest/) workflow to run the somatic mutation callers and SomaticSeq, created by [Afif Elghraoui](https://github.com/0xaf1f), is at [**utilities/snakemake**](utilities/snakemake).
* All the docker scripts have their corresponding singularity versions at utilities/singularities. They're created automatically with this [script](utilities/singularities/docker2singularity.py). They are not as extensively tested or optimized as the dockered ones. Read the pages at the dockered pipelines for descriptions and how-to's. Please let us know at Issues if any of them does not work.


## Video tutorial

This 8-minute video was created for SomaticSeq v1.0. The details are slightly outdated, but the main points remain the same.

  [![SomaticSeq Video](docs/SomaticSeqYoutube.png)](https://www.youtube.com/watch?v=MnJdTQWWN6w "SomaticSeq Video")
