# BPCells

BPCells is a package for high performance single cell analysis. It is designed to
cover the processing pipeline from ATAC fragments or RNA counts matrices through
to normalization, basic QC, and PCA. 

Three of the key distinguishing features that allow for high performance in BPCells are:
1. Bit-packing compression to allow for extremely compact storage of
   fragments or counts matrices on-disk or in memory.
2. C++ code that operates on all data in a streaming fashion to support low
   memory usage and efficient use of CPU cache.
3. Matrix-free SVD solvers in combination with implicit normalization calculations 
  to support computing the PCA of a normalized matrix while only ever storing the 
  original counts matrix in memory.

## Installation

BPCells is easiest to install directly from github:

```R
devtools::install_github("bnprks/BPCells")
```

Note that because of its use of SIMD extensions in C++, it is not yet compatible
with ARM architectures like the M1 Macs.

## Getting started

Two key principles to understand about using BPCells is that all operations are
*streaming* and *lazy*. 

Streaming means that only a minimal amount of data is 
stored in memory while a computation is happening. There is almost no
memory used storing intermediate results. Hence, we can compute operations 
on large matrices without ever loading them into memory.

Lazy means that no real work is performed on matrix or fragment objects until
the result needs to be returned as an R object or written to disk. This helps support
the streaming computation, since otherwise we would be forced to compute intermediate
results and use additional memory.

### Basic usage
We begin with a basic example of loading ATAC fragments from a 10x fragments file,
reading a peak set from a bed file, then calculating a cell x peak matrix.
```R
library("BPCells")

# File reading is lazy, so this is instantaneous
fragments <- open_10x_fragments("atac_fragments.tsv.gz")

# This is when we actually read the file, should take 1-2 minutes to scan
# since we bottleneck on gzip decompression.
packed_fragments <- write_packed_fragments(fragments)

# Important to set compress=FALSE for speed. Should take a few seconds
saveRDS(packed_fragments, "fragments.rds", compress=FALSE)

# Reloading from disk is only a few seconds now.
packed_fragments <- readRds("fragments.rds")

peaks <- read_tsv("peaks.bed", col_names=c("chr", "start", "end"), col_types="cii")

# This is fast because the peak matrix calculation is lazy.
# It will be computed on-the-fly when we actually need results from it.
# Note that we don't need to convert our peaks to 0-based coordinates since our bed file
# is already 0-based
peakMatrix <- overlapMatrix(packed_fragments, peaks, convert_to_0_based_coords=FALSE)

# Here is where the peak matrix calculation happens. Should take
# under 10 seconds.
R_matrix <- as(peakMatrix, "dgCMatrix")
```

### Streaming operations

The lazy, stream-oriented design means that we can calculate more complicated
transformations in a single pass. This is both faster and more memory-efficient
than calculating several intermediate results in a sequential manner.

As an example, we will perform the following pipeline:
1. Exclude fragments from non-standard chromosomes
2. Subset our cells
3. Add a Tn5 offset
4. Calculate the peakMatrix
5. Calculate the mean-accessibility per peak

If this were done using e.g. GRanges or sparse matrices, we would need to do 3
passes through the fragments while saving intermediate results, and 2 passes over
the peakMatrix.

With BPCell's streaming operations, this can all be done directly from the fragments in a single pass, and the memory
usage is limited to a few bytes per cell for iterating over the peakMatrix 
and returning the colMeans.
```R
# Here I make use of the pipe operator (%>%) for better readability
library("tidyverse")

# We'll subset to just the standard chromosomes
standard_chr <- which(str_detect(packed_fragments@chr_names, "^chr[0-9XY]+$"))

# Pick a random subset of 100 cells to consider
set.seed(1337)
keeper_cells <- sample(packed_fragments@cell_names, 100)

# Run the pipeline, and save the average accessibility per peak
peak_accessibility <- packed_fragments %>%
  select_chromosomes(standard_chr) %>%
  select_cells(keeper_cells) %>%
  shift_fragments(shift_start=4, shift_end=-5) %>%
  overlapMatrix(peaks) %>%
  colMeans()
```

Note that if we knew the cell names ahead of time, we could even perform this
operation directly on our orignal 10x fragments without ever saving the
fragments into memory. This would be fairly slow because 10x fragment files are
slow to decompress. With upcoming support for storing packed fragments directly
on disk, this can become a much faster operation without ever needing to store
fragments in memory.

## Roadmap

### Current support:
- Fragments
    - Reading/writing 10x fragment files on disk
    - Creation of packed fragment objects in memory
    - Interconversion of fragments objects with GRanges
    - Calculation of Cell x Peak matrices
- Matrices
    - Conversion to/from R sparse matrices
    - Multiplication by dense matrices or vectors
    - Calculation of statistics like rowSums, colSums, rowMeans, and colMeans

### Upcoming additions:
- Support for additional fragment formats:
    - Read/write packed fragments from disk (likely hdf5 format)
    - Read fragments from bam files
    - Support direct download of files from URLs
- Support for additional matrix formats:
    - Read/write hdf5 formats for 10X or AnnData matrices
    - Read/write packed counts matrices in memory or on disk
- Support for additional matrix normalizations:
    - ATAC-seq LSI
    - Seurat default normalization
    - sctransform normalization

### Performance goals/estimates
- Bit-packed storage:
    - Packed fragments are about 2x smaller than ArchR fragments and gzipped 10x fragment files
    - Packed fragments can be decompressed at >5GB/s, and so decoding is disk-limited on
      SSDs or RAID arrays while remaining competitive with reading directly from uncompressed 
      RAM
    - Packed matrices will probably be 4-6x smaller than the equivalent sparse matrices,
      with similarly excellent decompression speed expected
- PCA calculation:
    - Compared to Seurat's default normalization + PCA, BPCells will likely be about
      10x more efficient in memory and CPU. It is unclear if this will multiply with
      the 4-6x memory savings from bitpacking counts matrices.
    - LSI is not expected to be substantially faster than ArchR, although the C++
      implementation may provide a several-fold speedup, and bitpacking will provide
      a 4-6x memory savings.