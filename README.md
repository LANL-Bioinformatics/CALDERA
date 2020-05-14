# BIGSI++
A C++ reimplementation of the ultra-fast, [Bloom filter](https://en.wikipedia.org/wiki/Bloom_filter)-based sequence search originally developed by the [Iqbal group](https://www.nature.com/articles/s41587-018-0010-1). This project is similar in spirit to the Iqbal group's BIGSI follow-on project: [COBS: a Compact Bit-Sliced Signature Index](https://arxiv.org/abs/1905.09624). While the goals are very similar, the BIGSI++ project is an independent implementation of the original BIGSI algorithm with the addition of the following features:
1. Parallel, C++ implementation (using both OpenMP and MPI)
2. Adaptive Bloom filter size (similar to COBS)
3. Compressed Bloom filters
4. Bloom filter construction directly from [Sequence Read Archive](https://www.ncbi.nlm.nih.gov/sra) (SRA) files

The BIGSI++ pipeline has four primary components:
1. Streaming downloads of SRA files: `sra_download`
2. Streaming construction of Bloom filters from the downloaded SRA files: `bloomer`
3. Database construction from the Bloom filters: `build_db`
4. Searching the resulting database with a nucleic acid query: `bigsi++`

Please note that the Bloom filters created by the `bloomer` tool and ingested by the `build_db` are only needed during the database construction step. Once the database has been built, these temporary Bloom filters can be removed.

The following helper applications are also provided:
1. `dump_db`, a tool for dumping the header, annotation, and first few bit slices of a database file.
2. `check_bloom`, a tool for validating the checksum of a Bloom filter file.
3. `SriRachA`, a tool for per-read, kmer-based searching of SRA records against a set of user-provided query sequences. This tool is useful for confirming and investigating the per-accession matches reported by BIGSI++. This tool is contained in the `sriracha/` subdirectory (with a separate `Makefile` and `documentation`).

## Building and installing the code
The BIGSI++ code is written in C++ and requires the MPI (message passing interface; I recommend [OpenMPI](https://www.open-mpi.org/)). A compiler that supports OpenMP is also suggested, but not required.

Currently, downloading the SRA files is orchestrated by the `sra_download` program, with help from the `prefetch` tool (that is provided in the SRA toolkit) and a cluster scheduling system (so that Bloom filters can be created from SRA files while new files are being downloaded in parallel). The code currently assumes a [Torque/PBS](https://www.mcs.anl.gov/research/projects/openpbs/) compatible scheduler, but this is only for the current proof-of-principle implementation. As discussed below, downloading the entire SRA to a private network (i.e. non-[AWS](https://aws.amazon.com/), non-[Google](https://cloud.google.com/)) is **not** going to work -- downloading will take too long for the several petabytes of SRA data.

A cloud-compatible version of `sra_download` (coming soon!) will replace the Torque/PBS dependency with an AWS dependency to spawn additional AWS instances than compute Bloom filters from the downloaded files.

### Step 1: Download the SRA toolkit and associated APIs
In addition, the SRA toolkit and associated APIs are required if you would like to be able to download and/or construct Bloom filters from SRA files. In particular, you will need to download and build the following three GitHub projects from NCBI:
1. [ncbi-vdb](https://github.com/ncbi/ncbi-vdb)
2. [ngs](https://github.com/ncbi/ngs)
3. [sra-tools](https://github.com/ncbi/sra-tools)

Please note that these NCBI libraries are only needed if you will need to either download SRA files and/or create Bloom filters from SRA files. If you only need to either build a database from existing Bloom filters or search an existing database, then these NCBI libraries are *not* needed.

#### How to actually build the SRA toolkit packages
It can be a little tricky to compile the ncbi-vdb, ngs and sra-tools packages. Here is a step-by-step guide (with specific package install commands for a Red Hat-like OS that uses the `yum` package manager):

1. Install the `git` repository control software (e.g. `sudo yum install git`).
2. Install a c/c++ compiler (e.g. `sudo yum group install "Development Tools"`).
4. Install a Java development kit (while Java is not used for BIGSI++, a jdk is needed to compile the ncbi.ngs package).
5. Install libxml2 development libraries (e.g. `sudo yum install libxml2-devel.x86_64`). Many Linux distributions ship with libxml2 but not the development headers, which will be needed.
6. Create a directory to hold all of the NCBI SRA software (e.g. `mkdir SRA`). The rest of these instructions assume that the path to the SRA packages is `$HOME/SRA`.
7. Download sra-tools from GitHub (e.g. `git clone https://github.com/ncbi/sra-tools.git`).
8. Download ngs from GitHub (e.g. `git clone https://github.com/ncbi/ngs.git`).
9. Download ncbi-vdb from GitHub (e.g. `git clone https://github.com/ncbi/ncbi-vdb.git`).
10. Compile the NCBI packages in the **following order** (order is important!):

```
# Build and install ncbi-vdb
cd $HOME/SRA/ncbi-vdb
./configure --prefix=$HOME/SRA \
	--with-ngs-sdk-prefix=$HOME/SRA/ngs

# Before building on Amazon's AWS (and depending on choice of image/instance), you may need to remove the `-Wa,-march=generic64+sse4` flags from `ncbi-vdb/libs/krypto/Makefile` prior to running `make`
make
make install
```

```
# Build and install ngs/ngs-sdk (a sub-package of ngs)
cd $HOME/SRA/ngs/ngs-sdk
./configure --prefix=$HOME/SRA/ngs/ngs-sdk
make
make install
```

```
# Build and install ngs/ngs-java
cd $HOME/SRA/ngs/ngs-java
./configure --prefix=$HOME/SRA
make
make install
```

```
# Build and install ngs
cd $HOME/SRA/ngs
./configure --prefix=$HOME/SRA \
	--with-ncbi-vdb-prefix=$HOME/SRA/ncbi-vdb \
	--with-ngs-sdk-prefix=$HOME/SRA/ngs/ngs-sdk
make
make install
```

```
# Build and install sra-tools
cd cd $HOME/SRA/sra-tools
./configure --prefix=$HOME/SRA \
	--with-ngs-sdk-prefix=$HOME/SRA/ngs/ngs-sdk \
	--with-ncbi-vdb-sources=$HOME/SRA/ncbi-vdb

make
make install
```
11. Run the vdb configuration tool: `$HOME/SRA/bin/vdb-config --interactive`.
	- Select the `[X] report cloud instance identity` option
	- At this time, do *not* select the `[ ] accept charges for AWS`.
	- Select `Save`
	- Select `Exit`

The options you select here will depend on your local compute environment (e.g. cloud vs local).

### Step 2: Edit the BIGSI++ Makefile
After the three separate NCBI software packages have been downloaded and compiled, you will need to edit the BIGSI++ Makefile to specify the locations of both the include and library directories for the ncbi-vdb and ngs packages.

Edit the `SRA_LIB_PATH` variable to point to the directory on your system that contains the SRA library files.

Edit the `SRA_INCLUDE_PATH` variable to point to the directory on your system that contains the SRA include files.

### Step 3: Run `make -j`

Running `make` in the BIGS++ directory will build all pipeline components. There is no separate installation step required. If you encounter any problems, please email `jgans@lanl.gov` for assistance.

## SRA download

Streaming download of SRA files (i.e. download SRA file &rightarrow; create Bloom filter &rightarrow; remove SRA file) is performed by the `sra_download` program, whose command line arguments are:
```
Usage for SRA Download:
	-i <XML metadata file from ftp://ftp.ncbi.nlm.nih.gov/sra/reports/Metadata>
	--download <download directory for SRA data>
	--bloom <Bloom filter output directory>
	--log <log filename> (required to track progress for restarts)
	[-k <kmer length>] (default is 31)
	[-p <false positive probability (per k-mer, per-filter)>] (default is 0.25)
	[--min-kmer-count <minimum allowed k-mer count>] (default is 5)
	[--hash <hash function name>] (default is murmur)
		Allowed hash functions: murmur
	[--len.min <log2 Bloom filter len>] (default is 20)
	[--len.max <log2 Bloom filter len>] (default is 33)
	[--list (list, but do not download, SRA data)]
	[--sleep <sec> (time to sleep between downloads)]
	[--max-backlog <number of downloads> (Pause SRA downloading when exceeded)] (default is 25; 0 is no limit)
	[-t <number of download threads>] (default is 1)
	[--date.from <YYYY-MM-DD>] (only download SRA records received after this date)
	[--date.to <YYYY-MM-DD>] (only download SRA records received before this date)
	[--strategy <strategy key word>] (only download SRA records that match one of the specified experimental strategies)
	[--source <source key word>] (only download SRA records that match one of the specified exterimental sources)
```

- The argument to the `-i` flag is the compressed tar file of XML metadata dowloaded from the NCBI [FTP](tp://ftp.ncbi.nlm.nih.gov/sra/reports/Metadata) site. **Do not uncompress or untar this file!** Since the uncompressed/untarred files are large and cumbersome to manage, the `sra_download` program directly reads the SRA metadata and run accessions by uncompressing on the fly (and parses the tar file format to track the source files).
- The argument to the `--download` flag is the directory that will be used to temporarily stage SRA files during the download process and while the Bloom filters are being created by the `bloomer` tool. Ideally, this directory should be located on a fast storage device and must be visible to all of the backend node/cloud instances that will handle the conversion of SRA files into Bloom filters.
- The argument to the `--bloom` flag is the directory that will be used to store all of the Bloom filters.
- The argument of the `--log` flag is the filename of either a new log file to create (for a completely new run) or an existing log file (for restarting a download attempt or incrementally downloading newly available SRA files). This log file is a human readable text file that is also parsed by the `sra_download` program to identify which SRA files have already been downloaded.
- The argument to the `-k` flag is the integer k-mer length. The maximum allowed k-mer length is 32.
- The argument to the `-p` flag is the desired Bloom filter false positive rate (per k-mer and per-filter). This rate must be between 0 and 1, and it used to determine the Bloom filter parameters (i.e. number of bits and number of hash functions used per-k-mer). SRA files that have so many k-mers that the desired false positive rate threshold can not be obtained without exceeding the user defined limits of Bloom filter length and number of hash functions are skipped (and an empty Bloom filter file is written).
- The argument to the `--min-kmer-count` flag is the minimum number of times a given k-mer must be found in an SRA file. The purpose of this threshold is to "de-noise" (i.e. remove sequencing errors from) SRA files. Since sequencing errors can generate larg numbers of unique, but not biologically relevant k-mers, larger minimum k-mer counts significantly reduce the size of the Bloom filter needed to represent an SRA file.
- The argument to the `--hash` flag is the name of the hash function to use when constructing the Bloom filter. Currently, only the [murmur hash](https://github.com/aappleby/smhasher/blob/master/src/MurmurHash3.cpp) function is supported. In the future, additional hash functions may be added. I suspect that there is still room to improve hash function performance (i.e. increase speed, reduce collisions) for DNA sequences -- however, hash function construction is very tricky!
- The argument to the `len.min` flag is the log<sub>2</sub>(minimum Bloom filter length in bits) to test when computing the optimal Bloom filter length.
- The argument to the `len.max` flag is the log<sub>2</sub>(maximum Bloom filter length in bits) to test when computing the optimal Bloom filter length. Currently, the defaut maximum allowed Bloom filter length is 2<sup>33</sup> bits (which yields a maximum Bloom filter file size of approximately 1 GB).
- Specifing the `--list` flag will cause `sra_download` to print the SRA run accessions and metadata found in the XML input file and then exit **without** downloading any SRA files.
- The argument to the optional `--sleep` flag is the number of seconds to sleep after downloading an SRA file using the `prefetch` command. This is intended for situations where one might need to reduce the average bandwidth to NCBI SRA database.
- The argument to the `--max-backlog` flag sets the maximum number of SRA files that can be downloaded before being processed with the `bloomer` program to create Bloom filters (and to remove the SRA file). This is intended to prevent the exhaustion of local `--download` directory storage in the event that (a) the job scheduler has failed and/or (b) the number of compute nodes running the `bloomer` program can not keep up with the rate at which SRA files are being downloaded.
- The argument to the `-t` flag sets the integer number of threads that will be used to download SRA files in parallel (i.e. spawning independent instances of the `prefetch` program to download separate SRA run accessions). When downloading to a private (i.e. non-cloud) network, using upto 4 threads offers a bandwidth improvement.
- The argument to the `--date.from` flag restricts the download to SRA runs that were *received after* this date
- The argument to the `--date.to` flag restricts the download to SRA runs that were *received before* this date
- The argument to the `--strategy` flag restricts the download to SRA runs that have a matching experimental strategy. 
	- This command can be repeated multiple times.
	- The string matching is case sensitive.
	- A *partial* list of experimental strategies include:
		- RNA-Seq
		- WGS
		- AMPLICON
		- Bisulfite-Seq
- The argument to the `--source` flag restricts the download to SRA runs that have a matching experimental source. 
	- This command can be repeated multiple times.
	- The string matching is case sensitive.
	- A *partial* list of experimental sources include:
		- TRANSCRIPTOMIC
		- GENOMIC
		- METAGENOMIC
		- METATRANSCRIPTOMIC

Please note that downloading the *entire* SRA database is an enormous undertaking. There are several petabytes of data composed of millions of SRA run accessions in the SRA database, which is currently stored in the Amazona and Google clouds. The prefered usage is run this tool in the cloud to avoid additional data egress charges to NCBI (that are imposed by cloud providers when data is copied out of the cloud to a private network).

### SRA annotation information

Currently, SRA annotation information is extracted from the XML metadata dowloaded from the NCBI and included in the Bloom filter files and the final BIGSI++ database. Unless noted, information is stored as a string. Since not all fields are provided for every SRA record, the string "NA" indicates a missing field.
- Run
	- Accession
- Experiment
	- Accession
- Experiment
	- Title
	- Design description
	- Library name
	- Library strategy
	- Library source
	- Library selection
	- Instrument model
- Sample
	- Accession
	- Taxa
	- Attributes (as a collection of key/value pairs)
- Study
	- Accession
	- Title
	- Abstract

## Bloom filter construction

Bloom filter construction is handled by the `bloomer` program, which:
1. Reads sequence data directly from a local SRA file (downloaded using the NCBI SRA `prefetch` command) and an optional file of metadata (that is created by the `sra_download` program when it parses the XML inventory file provided by the SRA).
2. Extracts high frequency k-mers.
3. Selects the optimal Bloom filter parameters (i.e. the filter length in bits and the number of hash functions to apply to each k-mer).
4. Builds the Bloom filter in RAM.
5. Writes the Bloom filter and associated SRA metadata to storage (using a single file to store each Bloom filter). These Bloom filter files are temporary -- they are only used by the `build_db` program and are not intended to be directly searched by the user. After `build_db` has successfully ingested a Bloom filter file, the Bloom filter file can be deleted.

While the `bloomer` program is mostly intended to be a helper tool that is run by the `sra_download` program, it can be run manually. The command line arguments for the `bloomer` program are:
```
Usage for Bloomer (v. 0.3):
	-i <input SRA directory>
	-o <Bloom filter output file>
	[-k <kmer length>] (default is 31)
	[-p <false positive probability (per k-mer, per-filter)>] (default is 0.25)
	[--min-kmer-count <minimum allowed k-mer count>] (default is 5)
	[--hash <hash function name>] (default is murmur)
		Allowed hash functions: murmur
	[--no-meta (do not attempt to read the SRA meta-data file)]
	[--slice <num slice>] (number of SRA file slices to read in parallel; default is 5)
	[--len.min <log2 Bloom filter len>] (default is 20)
	[--len.max <log2 Bloom filter len>] (default is 33)
	[-v (turn on verbose output)]
```
- The argument to the `-i` flag is the input SRA directory, which contains:
	- The SRA file (with a ".sra" suffix)
	- All associated SRA datafiles (inluding reference sequences that are needed for SRA files which contain aligned reads).
	- The SRA metadata file (that is extracted from the NCBI SRA XML file by the `sra_download` program)
- The argument to the `-o` flag is the output file that will be written with the resulting Bloom filter an associated metadata.
- The argument to the `-k` flag is the integer k-mer length. The maximum allowed k-mer length is 32.
- The argument to the `-p` flag is the desired Bloom filter false positive rate (per k-mer and per-filter). This rate must be between 0 and 1, and it used to determine the Bloom filter parameters (i.e. number of bits and number of hash functions used per-k-mer). SRA files that have so many k-mers that the desired false positive rate threshold can not be obtained without exceeding the user defined limits of Bloom filter length and number of hash functions are skipped (and an empty Bloom filter file is written).
- The argument to the `--min-kmer-count` flag is the minimum number of times a given k-mer must be found in an SRA file. The purpose of this threshold is to "de-noise" (i.e. remove sequencing errors from) SRA files. Since sequencing errors can generate larg numbers of unique, but not biologically relevant k-mers, larger minimum k-mer counts significantly reduce the size of the Bloom filter needed to represent an SRA file.
- The argument to the `--hash` flag is the name of the hash function to use when constructing the Bloom filter. Currently, only the [murmur hash](https://github.com/aappleby/smhasher/blob/master/src/MurmurHash3.cpp) function is supported. In the future, additional hash functions may be added. I suspect that there is still room to improve hash function performance (i.e. increase speed, reduce collisions) for DNA sequences -- however, hash function construction is very tricky!
- The argument to the `len.min` flag is the log<sub>2</sub>(minimum Bloom filter length in bits) to test when computing the optimal Bloom filter length.
- The argument to the `len.max` flag is the log<sub>2</sub>(maximum Bloom filter length in bits) to test when computing the optimal Bloom filter length. Currently, the defaut maximum allowed Bloom filter length is 2<sup>33</sup> bits (which yields a maximum Bloom filter file size of approximately 1 GB).
- The `-v` flag turns on verbose output, which is useful for debugging.

Complex SRA records (i.e. metagenomic and large eukaryotic sequencing projects) can generate large numbers of k-mers, which must be counted so that we can discard the rare (i.e. low abundance) k-mers that are likely a result of sequencing errors. To enable stoarge of large k-mer sets in RAM, the `bloomer` program uses MPI to enable multiple computers to work together to distribute the memory requirements for storing the k-mers.

## Database construction

## Searching the database
