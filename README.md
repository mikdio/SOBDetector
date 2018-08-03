## 1. Theoretical background
There are two options to choose from when collecting and preserving tumor specimens for molecular analysis: fresh frozen (FF) or formalin-fixed paraffin-embedded (FFPE). While inserting a tissue into a phenol solution and fresh freezing it directly after resection results in an excellent stability, keeping it under -80 °C is indisputably more expensive, than embedding it into paraffin blocks after formalin fixation. This fixation process however, introduces artifactual mutations (mostly in the C>T direction due to the deamination of cytosine bases induced by formalin) into the DNA strands which results in a much lower quality. The major benefit of this ffpe procedure is that the specimen becomes no longer sensitive to heat, so it can be work with at room temperature.

Due to its cost effectiveness and simplicity, ffpe tissue processing remains the most common approach for tissue specimen storage. However, the presence of these artifactual mutations makes mutation calling (especially point mutation calling) a hard task. Over the past years, various amounts of tools have been created that have made the extraction of allele-specific segmentation data and copy number profiles from whole exome sequences (WES) possible (sequenza, ASCAT, …). However, in order to ensure this allele specificity these tools rely on heterogeneous positions, which can be strongly affected by the presence of these ffpe artifacts, making the final copy number estimations unreliable. Also, the well known somatic signatures (Alexandrov et al.) depend solely on the point mutations found in a tumor sample. It is imperative therefore, that before performing these analyses we ensure, that the mutations found by a variant caller are real, not mere artifacts.

Since formalin very likely affects only one of the strands (e.g. a C|G pair becomes T|G), a paired-end next generation sequencing approach can help in this additional filtering step. By counting not just the number of reads that support the alternative alleles, but the relative orientation of the reads as well (Forward-Reverse:FR or Reverse-Forward:RF), these ffpe artifacts will likely to have a strand orientation bias towards one of the directions, while true mutations should have approximately the same amount of FR and RF reads. This article is based on a tool: Strand Orientation Bias Detector (SOBDetector), that reanalyzes the mutations stored in a vcf by any mutation caller, and evaluates whether the reads which support the alternate alleles have a strand orientation bias using the original binary alignment (BAM) files. This method works only on Illumina-like sequencing approaches.

## 2. Method

During sequencing, we first fragment the DNA in (ideally) isotermal conditions. Then we ligate the adapter sequences, the primers, indices and sequences that are complementary to the oligose found in the flow cell to the ends of these fragments:
<div style="text-align:center; padding-top: 15px; padding-bottom: 15px; width:100%"><img src="./figures/paired_end_reads.svg" /></div>

Since both the Crick and Watson strands have read1s and read2s on them, after the alignment there sould be approximately the same amount of F1R2 and F2R1 "constellations" in the resulting binary alignment files:
<div style="text-align:center; padding-top: 15px; padding-bottom: 15px; width:100%"><img src="./figures/paired_end_reads2.svg"/></div>

Since the mutations induced by the storing process can be found only in one of the strands, they are likely to have a bias in the F1R2 and F2R1 orientation as well:

<div style="text-align:center; padding-top: 15px; padding-bottom: 15px; width:100%"><img src="./figures/paired_end_reads3.svg" /></div>

<div style="text-align:center; padding-top: 15px; padding-bottom: 15px; width:100%"><img src="./figures/paired_end_reads4.svg" /></div>

<strong>SOBDetector reanalyzes the mutations stored in a vcf by any mutation caller, and evaluates whether the reads which support the alternate alleles have a strand orientation bias using the original binary alignment (BAM) files.</strong> Although it was created for human somatic mutation filtration, it can be used on samples of any species and in a germline variant calling analysiss as well.

<div style="text-align:center; padding-top: 15px; padding-bottom: 15px; width:100%"><img src="./figures/figure_pipeline.png" /></div>

## 3 Requirements
SOBDEtector is a portable jar file, written in java 1.8. Apart of a java runtime environment, it only requires the presence of a relatively recent version of samtools (recommended v1.6 and above) in the system. 

## 4 Usage
The current version (v0.1) of SOBDetector can be used in two ways. 

### 4.1 By using a vcf file

The first is the regular one, when the user provides the location of a vcf file and its corresponding bam's. This can be implemented in the following way:

```bash
java -jar SOBDetector_v0.1.jar \ 
    --input-type VCF \ 
    --input-variants ./input-variants.vcf \ 
    --input-bam ./input.bam \ 
    --output-variants ./output.vcf \ 
    --only-passed true
```

In line 1 we provide SOBDetector for the java virtual machine. Optional java arguments such as the maximum amount of allowed memory or number of threads can be specified here, though multi-threading is not supported yet and the only thing that is stored in the heap is the vcf file itself, which is generally less than 100 Mb in size.
From line to we provide the mandatory arguments. These are the following:
 The type of the provided variant file. It can either be "VCF" or "Table" (we'll discuss the latter later).
 
* <strong> --input-variants: </strong>  Absolute or relative path that leads to the variant file (in this case, to a vcf).
* <strong> --inpupt-bam: </strong> Absolute or relative path of the binary alignment file, from which the variants were determined. It is important that in case the variant caller alters the original orientation of the reads while its running, for example by performing local de-novo assembly, we use the altered BAM file here.
* <strong> --output-variants </strong> Absolute or relative path to the desired output file. If the input is a vcf file, then this will be a vcf too. Note: all the folders must exist, the tool won't create them automatically.

The rest of the arguments are optional:

* <strong> --only-passed</strong> Defaults to false, if set true, variants that have non-PASSED FILTER attributes in the input vcf will be ignored. (It speeds up execution time.)
* <strong> --minBaseQuality</strong> Defaults to 0, the minimally considered base quality. If the quality of the considered base is less than this value, the read will be ignored.
* <strong> --minMappingQuality</strong> Defaults to 0, the minimally considered mapping quality. If the mapping quality of read is less than this value, then the read is ignored. 
\end{itemize}

### 4.2 By using a tabulated file

Many times the variants we are working with are collected into tab-delimited files (or R/pandas data frame). SOBDetector can work with this format too. All we have to do is to specify our intentions by setting the input-type to "Table" instead of "Vcf":

```bash
java -jar SOBDetector_v0.1.jar \ 
    --input-type Table \ 
    --input-variants ./input.table \ 
    --input-bam ./input.bam \ 
    --output-variants ./output.table \ 
```

Only the first four columns of such tab delimited files are mandatory. The necessary order of these attributes (using the standard vcf notation) has to be the following: 

| CHROM | POS | REF | ALT |

The file might contain other columns as well those will not be assessed. The output file of the analysis will have the same format as the input, containing the same columns, but additional attributes will be appended to them with the strand bias information. 

