# finemo_gpu

**FiNeMo** (**Fi**nding **Ne**ural network **Mo**tifs) is a GPU-accelerated hit caller for identifying occurrences of TFMoDISCo motifs within contribution scores generated by machine learning models.

## Installation

> **Note**
> This software is currently in development and will be available on PyPI once mature.
For now, we suggest installing it from source.

### Installing from Source

#### Clone the GitHub Repository

```sh
git clone https://github.com/austintwang/finemo_gpu.git
cd finemo_gpu
```

#### Create a Conda Environment with Dependencies

This step is optional but recommended

```sh
conda env create -f environment.yml -n $ENV_NAME
conda activate $ENV_NAME
```

#### Install the Python Package

```sh
pip install --editable .
```

#### Update an Existing Installation

To update, simply fetch the latest changes from the GitHub repository.

```sh
git pull
```

## Data Inputs

Required:

- Contribution scores for peak sequences in bigWig format, [ChromBPNet H5](https://github.com/kundajelab/chrombpnet/wiki/Generate-contribution-score-bigwigs#output-format) format, [BPNet H5](https://github.com/kundajelab/bpnet-refactor?tab=readme-ov-file#3-compute-importance-scores) format, or [tfmodisco-lite](https://github.com/jmschrei/tfmodisco-lite/tree/main#running-tfmodisco-lite) input format.
- Motif CWMs in [tfmodisco-lite](https://github.com/jmschrei/tfmodisco-lite/tree/main#running-tfmodisco-lite) H5 output format.

Recommended:

- Peak region coordinates in [ENCODE NarrowPeak](https://genome.ucsc.edu/FAQ/FAQformat.html#format12) format.

## Usage

FiNeMo includes a command-line utility named `finemo`.

### Preprocessing

The following commands transform input contributions and sequences into a compressed `.npz` file for quick loading. This file contains:

- `sequences`: A one-hot-encoded sequence array (`np.int8`) with dimensions `(n, 4, w)`, where `n` is the number of regions, and `w` is the width of each region. Bases are ordered as ACGT.
- `contribs`: A contribution score array (`np.float16`) with dimensions `(n, 4, w)` for hypothetical scores or `(n, w)` for projected scores only.

Preprocessing commands do not require GPU.

#### `finemo extract-regions-bw`

Extract sequences and contributions from FASTA and bigWig files.

> **Note** BigWig files only provide projected contribution scores.
Thus, the output only supports analyses based solely on projected contributions.

```console
usage: finemo extract-regions-bw [-h] -p PEAKS -f FASTA -b BIGWIGS [BIGWIGS ...] -o OUT_PATH [-w REGION_WIDTH]

options:
  -h, --help            show help message and exit
  -p PEAKS, --peaks PEAKS
                        A peak regions file in ENCODE NarrowPeak format. (*Required*)
  -f FASTA, --fasta FASTA
                        A genome FASTA file. If an .fai index file doesn't exist in the same directory, it will be created. (*Required*)
  -b BIGWIGS [BIGWIGS ...], --bigwigs BIGWIGS [BIGWIGS ...]
                        One or more bigwig files of contribution scores, with paths delimited by whitespace. Scores are averaged across files. (*Required*)
  -o OUT_PATH, --out-path OUT_PATH
                        The path to the output .npz file. (*Required*)
  -w REGION_WIDTH, --region-width REGION_WIDTH
                        The width of the input region centered around each peak summit. (default: 1000)
```

#### `finemo extract-regions-chrombpnet-h5`

Extract sequences and contributions from ChromBPNet H5 files.

```console
usage: finemo extract-regions-chrombpnet-h5 [-h] -c H5S [H5S ...] -o OUT_PATH [-w REGION_WIDTH]

options:
  -h, --help            show help message and exit
  -c H5S [H5S ...], --h5s H5S [H5S ...]
                        One or more H5 files of contribution scores, with paths delimited by whitespace. Scores are averaged across files. (*Required*)
  -o OUT_PATH, --out-path OUT_PATH
                        The path to the output .npz file. (*Required*)
  -w REGION_WIDTH, --region-width REGION_WIDTH
                        The width of the input region centered around each peak summit. (default: 1000)
```

#### `finemo extract-regions-bpnet-h5`

Extract sequences and contributions from BPNet H5 files.

```console
usage: finemo extract-regions-bpnet-h5 [-h] -c H5S [H5S ...] -o OUT_PATH [-w REGION_WIDTH]

options:
  -h, --help            show help message and exit
  -c H5S [H5S ...], --h5s H5S [H5S ...]
                        One or more H5 files of contribution scores, with paths delimited by whitespace. Scores are averaged across files. (*Required*)
  -o OUT_PATH, --out-path OUT_PATH
                        The path to the output .npz file. (*Required*)
  -w REGION_WIDTH, --region-width REGION_WIDTH
                        The width of the input region centered around each peak summit. (default: 1000)
```

#### `finemo extract-regions-modisco-fmt`

Extract sequences and contributions from tfmodisco-lite input `.npy`/`.npz` files.

```console
usage: finemo extract-regions-modisco-fmt [-h] -s SEQUENCES -a ATTRIBUTIONS [ATTRIBUTIONS ...] -o OUT_PATH [-w REGION_WIDTH]

options:
  -h, --help            show this help message and exit
  -s SEQUENCES, --sequences SEQUENCES
                        A .npy or .npz file containing one-hot encoded sequences. (*Required*)
  -a ATTRIBUTIONS [ATTRIBUTIONS ...], --attributions ATTRIBUTIONS [ATTRIBUTIONS ...]
                        One or more .npy or .npz files of hypothetical contribution scores, with paths delimited by whitespace. Scores are averaged across files. (*Required*)
  -o OUT_PATH, --out-path OUT_PATH
                        The path to the output .npz file. (*Required*)
  -w REGION_WIDTH, --region-width REGION_WIDTH
                        The width of the input region centered around each peak summit. (default: 1000)
```

### Hit Calling

#### `finemo call-hits`

Identify hits in input regions using TFMoDISCo CWM's.

```console
usage: finemo call-hits [-h] [-M {hp,pp,ph,hh}] -r REGIONS -m MODISCO_H5 [-p PEAKS] [-C CHROM_ORDER] -o OUT_DIR [-t CWM_TRIM_THRESHOLD] [-a ALPHA] [-f] [-s STEP_SIZE] [-A STEP_ADJUST] [-c CONVERGENCE_TOL] [-S MAX_STEPS] [-b BATCH_SIZE] [-d DEVICE]

options:
  -h, --help            show help message and exit
  -M {hh,pp,ph,hp}, --mode {hh,pp,ph,hp}
                        The type of attributions to use for CWM's and input contribution scores, respectively. 'h' for hypothetical and 'p' for projected. (default: pp)
  -r REGIONS, --regions REGIONS
                        A .npz file of input sequences and contributions, created with a `finemo extract-regions-*` command. (*Required*)
  -m MODISCO_H5, --modisco-h5 MODISCO_H5
                        A tfmodisco-lite output H5 file of motif patterns. (*Required*)
  -p PEAKS, --peaks PEAKS
                        A peak regions file in ENCODE NarrowPeak format, exactly matching the regions specified in `--regions`. If omitted, outputs will lack absolute genomic coordinates. (Optional)
  -C CHROM_ORDER, --chrom-order CHROM_ORDER
                        A tab-delimited file with chromosome names in the first column to define sort order of chromosomes. Missing chromosomes are ordered as they appear in -p/--peaks. (Optional)
  -I MOTIFS_INCLUDE, --motifs-include MOTIFS_INCLUDE
                        A tab-delimited file with tfmodisco motif names (e.g pos_patterns.pattern_0) in the first column to include in hit calling. If omitted, all motifs in the modisco H5 file are used. (Optional)
  -o OUT_DIR, --out-dir OUT_DIR
                        The path to the output directory. (*Required*)
  -t CWM_TRIM_THRESHOLD, --cwm-trim-threshold CWM_TRIM_THRESHOLD
                        The threshold to determine motif start and end positions within the full CWMs. (default: 0.3)
  -a ALPHA, --alpha ALPHA
                        The L1 regularization weight. (default: 0.7)
  -f, --no-post-filter  Do not perform post-hit-calling filtering. By default, hits are filtered based on a minimum correlation of `alpha` with the input contributions. (default: False)
  -s STEP_SIZE-MAX, --step-size-max MAX-STEP_SIZE
                        The maximum optimizer step size. (default: 3.0)
  -i STEP_SIZE-MIN, --step-size-min MIN-STEP_SIZE
                        The maximum optimizer step size. (default: 0.08)
  -A STEP_ADJUST, --step-adjust STEP_ADJUST
                        The optimizer step size adjustment factor. If the optimizer diverges, the step size is multiplicatively adjusted by this factor (default: 0.7)
  -c CONVERGENCE_TOL, --convergence-tol CONVERGENCE_TOL
                        The tolerance for determining convergence. The optimizer exits when the duality gap is less than the tolerance. (default: 0.0005)
  -S MAX_STEPS, --max-steps MAX_STEPS
                        The maximum number of optimization steps. (default: 10000)
  -b BATCH_SIZE, --batch-size BATCH_SIZE
                        The batch size used for optimization. (default: 2000)
  -d DEVICE, --device DEVICE
                        The pytorch device name to use. Set to `cpu` to run without a GPU. (default: cuda)
```

#### Outputs

`hits.tsv`: The full list of coordinate-sorted hits with the following fields:

- `chr`: Chromosome name. `NA` if peak coordinates (`-p/--peaks`) are not provided.
- `start`: Hit start coordinate from trimmed CWM, zero-indexed. Absolute if peak coordinates are provided, otherwise relative to the input region.
- `end`: Hit end coordinate from trimmed CWM, zero-indexed, exclusive. Absolute if peak coordinates are provided, otherwise relative to the input region.
- `start_untrimmed`: Hit start coordinate from trimmed CWM, zero-indexed. Absolute if peak coordinates are provided, otherwise relative to the input region.
- `end_untrimmed`: Hit end coordinate from trimmed CWM, zero-indexed,exclusive. Absolute if peak coordinates are provided, otherwise relative to the input region.
- `motif_name`: The hit motif name as specified in the provided tfmodisco H5 file.
- `hit_coefficient`: The regression coefficient for the hit. Values are normalized per peak region. This is the primary hit score.
- `hit_coefficient_global`: The regression coefficient for the hit, scaled by the overall importance of the region.
- `hit_correlation`: The correlation between the untrimmed CWM and the contribution score of the motif hit.
- `hit_importance`: The total absolute contribution score within the motif hit.
- `strand`: The orientation of the hit (`+` or `-`).
- `peak_name`: The name of the peak region containing the hit, taken from the `name` field of the input peak data. `NA` if `-p/--peaks` is not provided.
- `peak_id`: The numerical index of the peak region containing the hit.

`hits_unique.tsv`: A deduplicated list of hits in the same format as `hits.tsv`. In cases where peak regions overlap, `hits.tsv` may list multiple instances of a hit, each linked to a different peak. `hits_unique.tsv` arbitrarily selects one instance per duplicated hit. This file is generated only if `-p/--peaks` is specified.

`hits.bed`: A coordinate-sorted BED file of unique hits, generated only if `-p/--peaks` is provided. It includes:

- `chr`: Chromosome name.
- `start`: Hit start coordinate from trimmed CWM, zero-indexed.
- `end`: Hit end coordinate from trimmed CWM, zero-indexed, exclusive.
- `motif_name`: Hit motif name, taken from the provided tfmodisco H5 file.
- `score`: The `hit_correlation` score, multiplied by 1000 and cast to an integer.
- `strand`: The orientation of the hit (`+` or `-`).

`peaks_qc.tsv`: Per-peak statistics. It includes:

- `peak_id`: The numerical index of the peak region.
- `nll`: The final regression negative log likelihood, proportional to the mean squared error (MSE).
- `dual_gap`: The final duality gap.
- `num_steps`: The number of optimization steps taken.
- `step_size`: The optimization step size.
- `global_scale`: The peak-level scaling factor, used to normalize by overall importance.
- `chr`: The chromosome name, omitted if `-p/--peaks` not provided.
- `peak_region_start`: The start coordinate of the peak region, zero-indexed, omitted if `-p/--peaks` not provided.
- `peak_name`: The name of the peak region, derived from the input peak data's `name` field, omitted if `-p/--peaks` not provided.

`params.json`: The parameters used for hit calling.

#### Additional notes

- The `-a/--alpha` is the primary hyperparameter to tune, where higher values result in fewer but more confident hits. This parameter essentially represents the highest expected correlation between a CWM and a non-informative background signal. Values typically fall between 0.5 and 0.8.
- The `-t/--cwm-trim-threshold` parameter determines the threshold for trimming CWMs. If you find that motif flanks are being trimmed too aggressively, consider lowering this value. However, a too-high value may result in closely-spaced motif instances being missed.
- Set `-b/--batch-size` to the largest value your GPU memory can accommodate. **If you encounter GPU out-of-memory errors, try lowering this value.**
- Legacy TFMoDISCo H5 files can be updated to the newer TFMoDISCo-lite format with the `modisco convert` command found in the [tfmodisco-lite](https://github.com/jmschrei/tfmodisco-lite/tree/main) package.

### Output reporting

#### `finemo report`

Generate an HTML report (`report.html`) visualizing TF-MoDISCo seqlet recall and hit distributions.
The input regions must have genomic coordinates and match exactly those used during the TF-MoDISCo motif discovery process.
This command does not utilize the GPU.

```console
usage: finemo report [-h] -r REGIONS -H HITS -p PEAKS -m MODISCO_H5 -o OUT_DIR [-W MODISCO_REGION_WIDTH]

options:
  -h, --help            show this help message and exit
  -r REGIONS, --regions REGIONS
                        "A .npz file containing input sequences and contributions. Must be the same as those used for motif discovery and hit calling. (*Required*)
  -H HITS, --hits HITS  The `hits.tsv` output file generated by the `finemo call-hits` command on the regions specified in `--regions`. (*Required*)
  -p PEAKS, --peaks PEAKS
                        A file of peak regions in ENCODE NarrowPeak format, exactly matching the regions specified in `--regions`. (*Required*)
  -m MODISCO_H5, --modisco-h5 MODISCO_H5
                        The tfmodisco-lite output H5 file of motif patterns. Must be the same as that used for hit calling unless `--no-recall` is set. (*Required*)
  -o OUT_DIR, --out-dir OUT_DIR
                        The path to the output directory. (*Required*)
  -W MODISCO_REGION_WIDTH, --modisco-region-width MODISCO_REGION_WIDTH
                        The width of the region around each peak summit used by tfmodisco-lite. (default: 400)
  -t CWM_TRIM_THRESHOLD, --cwm-trim-threshold CWM_TRIM_THRESHOLD
                        The threshold to determine motif start and end positions within the full CWMs. This should match the value used in `finemo call-hits`. (default: 0.3)
  -n, --no-recall       Do not compute motif recall metrics. (default: False)
```
