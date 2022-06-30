# scUTRquant
A bioinformatics pipeline for single-cell 3' UTR isoform quantification.

# Overview
The **scUTRquant** pipeline builds on `kallisto bus` to provide a reusable tool for 
quantifying 3' UTR isoforms from 3'-end tag-based scRNA-seq datasets. The pipeline
is based on Snakemake and provides both Conda and Docker images to satisfy software 
requirements. It includes prebuilt reference UTRomes for **hg38** and **mm10**, which ensures a 
consistent set of features (3' UTR isoforms) across different runs. In total, this provides 
a rapid pipeline for recovering 3' UTR isoform counts from common scRNA-seq datasets.

## Inputs
The pipeline takes as input:

- set of FASTQ or BAM (CellRanger output) files from scRNA-seq experiments
- *target* transcriptome defined by:
  - kallisto index of UTRome (**hg38** and **mm10** provided)
  - GTF annotation of UTRome (**hg38** and **mm10** provided)
  - TSV merge annotation (**hg38** and **mm10** provided)
- YAML configuration file that controls pipeline parameters
- CSV sample sheet detailing the FASTQ/BAM files to be processed
- barcode whitelist (optional)
- CSV of cell annotations (optional)

**Note on UTRome Index**

The pipeline includes code to download prebuilt **hg38** and **mm10** UTRome GTFs and
kallisto indices. These prebuilt indices were generated by augmenting the protein coding
transcripts that have verified 3' ends in the GENCODE v39 and vM21 annotations with
high-confidence cleavage sites called from the Human Cell Landscape and Mouse Cell Atlas
datasets. These augmented transcriptomes were then truncated to include only the last
500 nts of each transcript and then deduplicated. Finally, the merge file contains
information on transcripts whose cleavage sites differ by fewer than 200 nts, which
corresponds to the empirical resolution limit for `kallisto` quantification as
determined by simulations.

Please see [our accompanying manuscript][ref:scUTRquant] for more details.

## Outputs
The primary output of the pipeline is a Bioconductor `SingleCellExperiment` object.
The `counts` in the object is a sparse `Matrix` of 3' UTR isoform counts; the `rowRanges` 
is a `GenomicRanges` of the 3' UTR isoforms; the `rowData` is a `DataFrame` with additional
information about 3' UTR isoforms; and the `colData` is a `DataFrame` populated with sample 
metadata and optional user-provide cell annotations.

To assist users in quality control, the pipeline additionally generates HTML reports 
for each sample.

The pipeline is configured to retain intermediate files, such as BUS and MTX files.
Advanced users can readily customize the pipeline to only generate the files they 
require. For example, users who prefer to work with alternative scRNA-seq data structures,
such as those used in Scanpy or Seurat, may wish to terminate the pipeline at MTX 
generation.

# Setup

## Requirements
The pipeline can use either Conda/Mamba or Singularity to provide the required software.

### Conda/Mamba Mode (MacOS or Linux)
Snakemake can use Conda to install the needed software. This configuration requires:

 - [Snakemake][ref:snakemake] >= 5.11<sup>ª</sup>
 - [Conda](https://docs.conda.io/projects/conda/en/latest/)

If Conda is not already installed, we strongly recommend installing 
[Mambaforge](https://github.com/conda-forge/miniforge#mambaforge). If Conda was previously
installed, strongly recommend installing Mamba:

```bash
conda install -n base -c conda-forge mamba
```
 
### Singularity Mode (Linux)
Snakemake can use the pre-built scUTRsquant Docker image to provide all additional software.
This configuration requires installing:

 - [Snakemake][ref:snakemake] >= 5.11<sup>ª</sup>
 - [Singularity](https://singularity.lbl.gov/index.html)


<sub>**[a]**: Snakemake v7.8.0-7.8.3 enforced a Conda configuration setting of `channel_priority: strict` by raising an exception. However, `scUTRquant` uses environments that require `channel_priority: flexible` to properly solve. Snakemake v7.8.4+ will warn against this, but can safely be ignored.</sub>

## Installation
1. Clone the repository.
    ```
    git clone git@github.com:Mayrlab/scUTRquant.git
    ```

2. Download the UTRome annotation, kallisto index, and merge file.
    **Human (hg38)**
    ```
    cd scUTRquant/extdata/targets/utrome_hg38_v1
    sh download_utrome.sh
    ```
    **Mouse (mm10)**
    ```
    cd scUTRquant/extdata/targets/utrome_mm10_v1
    sh download_utrome.sh
    ```
    **Reuse Tip:** For use across multiple projects, it is recommended to centralize 
    these files and change the entries in the `configfile` for `utrome_gtf`,
    `utrome_kdx`, and `utrome_merge` to point to the central location. In that
    case, one does not need to redownload the files.

3. (Optional) Download the barcode whitelists.
    ```
    cd scUTRquant/extdata/bxs
    sh download_10X_whitelists.sh
    ```
    **Reuse Tip:** Similar to the UTRome files, these can also be centralized
    and referenced by the `bx_whitelist` variable in the `configfile`.

# Running Examples
Examples are provided in the `scUTRquant/examples` folder. Each includes a script
for downloading the raw data, a `sample_sheet.csv` formatted for use in the pipeline,
and a `config.yaml` file for running the pipeline.

Look for output files in the `qc/` and `data/` folders.

Note that the `config.yaml` uses paths relative to the `scUTRquant` folder.

## 1K Neurons (10xv3) - BAM

1. Download the raw data.
    ```
    cd scUTRquant/examples/neuron_1k_v3_bam/
    sh download.sh
    ```

2. Run the pipeline.

   **Conda Mode**
    ```
    cd scUTRquant
    snakemake --use-conda --configfile examples/neuron_1k_v3_bam/config.yaml
    ```

   **Singularity Mode**
    ```
    cd scutr-quant
    snakemake --use-singularity --configfile examples/neuron_1k_v3_bam/config.yaml
    ```

## 1K Heart (10xv3) - FASTQ

1. Download the raw data.
    ```
    cd scUTRquant/examples/heart_1k_v3_fastq/
    sh download.sh
    ```

2. Run the pipeline.

   **Conda Mode**
    ```
    cd scUTRquant
    snakemake --use-conda --configfile examples/heart_1k_v3_fastq/config.yaml
    ```

   **Singularity Mode**
    ```
    cd scUTRquant
    snakemake --use-singularity --configfile examples/heart_1k_v3_fastq/config.yaml
    ```


# File Specifications
## Configuration File

The Snakemake `configfile` specifies the parameters used to run the
pipeline. The following keys are expected:

 - `dataset_name`: name used in the final `SingleCellExperiment` object
 - `tmp_dir`: path to use for temporary files
 - `sample_file`: CSV-formatted file listing the samples to be processed
 - `sample_regex`: regular expression used to match sample IDs; including a specific
     regex helps to constrain Snakemake's DAG-generation
 - `output_type`: a list of outputs, including `"txs"` and/or `"genes"`
 - `target`: name of target(s) to which to pseudoalign; valid targets are defined in 
     the `extdata/targets/targets.yaml`; multiple targets can be specified in list format
 - `tech`: argument to `kallisto bus` indicating the scRNA-seq technology; see
     [the documentation](https://pachterlab.github.io/kallisto/manual#bus) for supported values
 - `strand`: argument to `kallisto bus` indicating the orientation of sequence reads
     with respect to transcripts; all 10X 3'-end libraries use `--fr-stranded`;
     omitting this argument eliminates the ability to correctly assign reads to
     transcripts when opposing stranded genes overlap
 - `bx_whitelist`: file of valid barcodes used in `bustools correct`
 - `min_umis`: minimum number of UMIs per cell; cells below this threshold are excluded
 - `cell_annots`: (optional) CSV file with a `cell_id` entry that matches thee `<sample_id>_<cell_bx>` format
 
### Default Values

Snakemake can draw values for `config` in three ways:

 1. `scUTRquant/config.yaml`: This file is listed as the `configfile` in the Snakefile. 
 2. `--configfile config.yaml`: The file provided at the commandline.
 3. `--config argument=value`: A specific value for an argument 
 
This list runs from lowest to highest precedence. Configuration values that do not differ from those in `scUTRquant/config.yaml` can be left unspecfied in the YAML given by the `--configfile` argument. That is, one can use the `scUTRquant/config.yaml` to define shared settings, and only list dataset-specific config values in the dataset's YAML.

## Sample File

The `sample_file` provided in the Snakemake configuration is expected to be a CSV
with at least the following columns:

 - `sample_id`: a unique identifier for the sample; used in file names and paths
     of intermediate files derived from the sample
 - `file_type`: indicates whether sample input is `'bam'` or `'fastq'` format
 - `files`: a semicolon-separated list of files; for multi-run (e.g., multi-lane)
     samples, the files must have the order:
     ```
     lane1_R1.fastq;lane1_R2.fastq;lane2_R1.fastq;lane2_R2.fastq;...
     ```

## Targets File
The `extdata/targets.yaml` defines the targets available to pseudoalign to. The default configuration provides `utrome_mm10_v1`, but additional entries can be added. A target has the following fields:

 - `path`: location where files are relative to
 - `genome`: genome identifier (e.g., `mm10`)
 - `gtf`: GTF annotation of UTRome; used in annotating rows
 - `kdx`: Kallisto index for UTRome
 - `merge`: TSV for merging features (isoforms)
 - `tx_annots`: (optional) RDS file containing Bioconductor DataFrame object with annotations for transcripts
 - `gene_annots`: (optional) RDS file containing Bioconductor DataFrame object with annotations for genes


# Customization
## Creating Custom Targets
The [Bioconductor package `txcutr`][ref:txcutr] provides methods for generating truncated transcriptome 
annotations. The [txcutr-db repository](https://github.com/Mayrlab/txcutr-db) provides an example Snakemake 
pipeline for using `txcutr` to generate the files needed for the custom target, starting from Ensembl or 
GENCODE annotations.

> **Recommendations:** For 10X Chromium 3' Single Cell libraries, we use a 500 nt truncation length
> and a 200 nt merge distance (details in [the sqUTRquant manuscript][ref:scUTRquant]). We recommend
> filtering transcripts to only protein-coding transcripts with validated 3' ends. For GENCODE, this
> means requiring `transcript_type "protein_coding"` and excluding transcripts with the tag **mRNA_end_NF**.

Once the GTF, KDX, and TSV files are generated, we recommend placing them in a new folder under `extdata/targets`.
Then, edit the `extdata/targets/targets.yaml` to include a new entry. This will look something like:

**extdata/targets/targets.yaml**
```yaml
custom_target_name:
  path: "extdata/targets/custom_target_name/"
  genome: "mm10"
  gtf: "custom_target.gtf"
  kdx: "custom_target.kdx"
  merge_tsv: "custom_target.merge.tsv"
  tx_annots: null
  gene_annots: null
```

Note that the pipeline supports running on multiple targets in parallel. This can be done in the  
configuration file, like so,

**config.yaml**
```yaml
target:
  - target1
  - target2
# ...
```

Or, if specifying targets at Snakemake invocation, one would use the syntax:

```bash
snakemake --config target='[target1,target2]' # ...
```

## Snakemake Cluster Profiles
The rules in the `Snakefile` include `threads` and `resources` arguments per rule. These values are compatible for use with [Snakemake profiles](https://github.com/Snakemake-Profiles) for cluster deployment. The current defaults will attempt to use up to 16 threads and 16GB of memory in the `kallisto bus` step. Please adjust to fit the resources available on the deployment cluster. We recommend that cluster profiles include both `--use-singularity` and `--use-conda` flags by default. Following this recommendation, an example run, for instance on **neuron_1k_v3_fastq**, with profile name `profile_name`, would take the form:

```bash
snakemake --profile profile_name --configfile examples/neuron_1k_v3_fastq/config.yaml
```


# Citation

Fansler, M. M., Zhen, G., & Mayr, C. (2021). Quantification of alternative 3′UTR isoforms from single cell RNA-seq data with scUTRquant. *BioRxiv*, 2021.11.22.469635. https://doi.org/10.1101/2021.11.22.469635


<!-- References -->

[ref:scUTRquant]: https://doi.org/10.1101/2021.11.22.469635
[ref:snakemake]: https://snakemake.readthedocs.io/en/stable/index.html
[ref:txcutr]: https://doi.org/doi:10.18129/B9.bioc.txcutr
