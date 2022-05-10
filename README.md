


# SYNPHONI

SYNPHONI aims to find sets of genes that were located in close proximity to each other in a given ancestor (syntenic blocks). 

Here, the term synteny is meant according to the definition in [Renwick, 1971](https://doi.org/10.1146/annurev.ge.05.120171.000501), i.e. genes located on the same chromosome in two or more species, regardless of conserved order or distance. This is also referred to as conserved macrosynteny (see [Simakov et *al.* 2022](https://doi.org/10.1126/sciadv.abi5884)). Of note, the Ancestral syntenic blocks identified by SYNPHONI are closer to the definition made in [Pevzner and Tesler 2003](https://doi.org/10.1101/gr.757503), where microrearrangements are allowed within syntenic blocks, but macrorearrangements are not. 

All the descendants of this ancestral microsyntenic blocks are recovered all extant genomes where the ancestral synteny has been retained, even if ancestral proximity has not.

Accordingly, SYNPHONI has been designed to the study of transformation of ancient syntenic blocks across scales of genome organization (for examples, please see the manuscript associated with SYNPHONI). Furthermore, since all the block sets are inferred for each node of the taxonomic sample independently, it is also possible to observe how a given block evolved in the different lineages of a tree.

# Getting started

For now, SYNPHONI runs on a single core. A sample of 80 metazoans was processed in less than 48 hours using a single core.

## Requirements

SYNPHONI requires python 3.9 or higher and depends on the following python libraries to be installed:
* networkx version 2.6.2
* ete3 version 3.1.2
* numpy version 1.21.2
* scipy version 1.7.1

## Inputs and outputs

A SYNPHONI run requires 4 main steps to be run, each specified using the `Step.*.py scripts`.
In the future, I might create a workflow manager to make it easier to run.

**Before starting**, check that you have all the inputs prepared:

* The positions of the proteins on the genomes (one isoform per gene), chrom format including species prefix
* An orthology file where the protein names start with prefix
* A newick tree (branch lengths do not matter, only the topology) with named nodes, where the tips are species prefix.

The output of SYNPHONI, for a given node will be:

* A *synt* file, containing informations about the blocks (species, genome location, genes contained, orthogroups of these genes).
* A *clusters* file, displaying which multispecies block the blocks of the *synt file* belong to.

Descriptions of the **.clus**, **.chrom**, and **synt/clusters** format, as well as scripts to convert between gff or bed to chrom or Orthofinder HOGs to clus format can be found [Here](https://github.com/nsmro/comparative_genomics_utils/tree/main/Microsynteny)


## Example of a SYNPHONI run



SYNPHONI is composed of 4 steps:

* **Step1**: Build the dist file, with all the minimum distances between any pair of OGs found in two species or more
* **Step2**: Estimate ancestral distances in specified nodes (ancestral OG graphs), based on the cladogram provided by the user
* **Step3**: Find sets of Orthogroups (OGs), which correspond to approximations of ancestral syntenic blocks in the ancestral OG graphs
* **Step4**: Recover ancestral syntenic blocks in extant species (outputs synt and clusters files) 

### Step1 - build a dist file

This step recovers the minimum distances (intevening genes) between any syntenic pair of OGs found in two or more species.All the possible orthogroup combinations will be examined for all the species. Since the number of genes in metazoans is more or less constant, adding more species should lead to a linear increase of the runtime.

**Input files:**
* orthology file (clus format)
* locations of genes (chrom format)

#### Command for example data
The command to run step1 on the example data is:
```bash
./step1_make_dists.py -c example_data/dataset_test.clus $(ls example_data/chrom_files/*chrom) -o myresults
```

This is the longest step of the pipeline (should take 3-5 minutes on the example data)  and creates a folder "myresults" containing two files:

* `myresults.dist`:  the matrix

* `chromdata.pickle`: a pickle file containings the parsed/filtered genome locations (used in step4)

#### Step1 - parameters:

* **-c, --clusfile**: orthology file, in clus format
* **-o, --output**: name of the output folder
* **-m, --maxpara**: all the orthogroups where one species posessed more paralogs that specified will be excluded from all downstream analysis (default: 100)
* **-s, --species_threshold**: number of species a pair of OGs need to be found syntenic in to be retained (default: 2).

It's more efficient to leave species threshold set up at 2, and filter more stringently in the following steps, especially since step1 has the longest computational time.

### Step2 - infer ancestral OG networks
This step will estimate the ancestral distance between the OG pairs in all the specified ancestors, generating an edgelist of the ancestral OG networks

**Inputs file:**
* species cladogram (newick format)
* dist file (created by step1)

For a given node of interest (passed to `-n` argument) we defined the five following taxonomic groups:
- Ingroup = all species descended from a given N
- Sister group = all the species of the clade with the closest relationship to the ingroup 
- Outgroup = all species outside of a given ingroup
- Ingroup taxon = all species of a children clade within a given ingroup
- Outgroup taxon = all species of a children clade within a given outgroup 

#### Command for example data
The command to run step2 on the example data for the Nephrozoa Last Common Ancestor is:
```bash
./step2_filter_pairs.py -d myresults/myresults.dist -s example_data/dataset_test.tre -n Nephrozoa
```
This should take up to a minute to process the example dataset, and will create a the `myresults.m_3.Nephrozoa.dist` file in the current working directory (file is named based on dist file name, species_threshold and node_name).
While it is possible to pass multiple node names to a single step2 command, I'd recomment to limit each step2 command to a single node, and run the different commands in parallel.

#### Step2 - parameters:
* **-s, --species_tree**: tree of the prefix name (should match the field names of the third column and onward in the `myresults.dist` file created by step1)
* **-n, --node_names**: space-separated of the nodes for which you want to build ancestral OGs networks.
* **-m, --species_threshold** (default: 3): How many species of a taxonomic group should posess the pair to be considered populated. At least two taxonomic groups are required to be populated (two ingroup taxa, one ingroup taxon and at least one sister group taxon or one ingroup taxon and the outgroup). If the threshold is larger than the taxonomic group considered, all the species of the groupe are required to posess the syntenic OG pair
* **-d, --dist**: dist file created by step1

The m parameter will determine how many orthogroup syntenic relationship will be included in the ancestral OG network. A high species_threshold (larger than all ingroups taxa) means that monophyly of the syntenic relationships is required (not a reasonable assumption, see [Winter et *al.*, 2016](https://doi.org/10.1093/nar/gkw843)). A species threshold of one on the other hand will lead to the detection of a lot of noise (e.g. convergent rearrangements, misassigned orthologs).

### Step3 - Isolate OG sets from the ancestral OG networks
This step will isolate the OGs composing ancestral syntenic blocks for a given node. It relies on using a dynamically determined nmax to isolate subgraphs of the ancestral OG graph. All the edges above this estimated nmax will be deleted from the graph, leaving only connected components (which are further refined, see associate dmanuscript for details).

**Inputs file:**
* node-specific dist file generated with step 2

#### Finding nmax automatically with the `step2.5_optimal_nmax.py` script
There is a script to estimate nmax provided in tools, but it relies on a folder structure (`node_name/m_*`) to determine node name and species_threshold used for the node-specific dist files.

This command will create a nested folder structure into a folder called `bynode`, move the dist file into it, and run the script to estimate the nmax:
The directory where the nested structure is provided as a positional argument (`bynode`), the `-o, --output` option takes as argument the prefix of the output files.
```bash
mkdir -p myresults/bynode/Nephrozoa/m_3; mv myresults.m_3.Nephrozoa.dist  myresults/bynode/Nephrozoa/m_3
tools/step2.5_optimal_nmax.py myresults/bynode/ -o myresults/nmax_test
```
Two files are created with this command (within the `myresults` folder):
* `nmax_test.tsv.component.csv`: table (csv format) comprising the size of the largest connected component with varying nmax thresholds, for all the nodes and associated species_threshold parameters found in `bynode`. Columns are as follows:
    * **node**: name of the node (gathered from the folder names)
    * **m**: species_threshold used to generate the ancestral OG graph (gathered from the folder names)
    * **nmax**: all the values nmax took (start, stop, step are three spece separated arguments passed to `-n, --nmax` option. Defaults are 1 100 1, i.e. all the values between 1 and 100 with increments of 1)
    * **proportion**: proportion of all the vertices of the ancestral OG graph within the largest component of the graph
* `nmax_test.tsv.inflection.csv`: table (csv format) comprising the suggested nmax (x-left column) for each node/species_threshold pair. Columns of the table are as follows:
    * **node**: name of the node (gathered from the folder names)
    * **m**: species_threshold used to generate the ancestral OG graph (gathered from the folder names)
    * **x-left**: estimated optimal nmax. The size of the largest component as a function of the largest nmax follows a sigmoid curve (increasing curve, i.e. convex portion, then concave portion). The estimated nmax is the elbow of the convex portion of the curve (see the associated manuscript for details).
 
 In the example data, if `node=Nephrozoa` and `species_threshold=3`, the estimated nmax is 52.
 This means that all the edges of the ancestral OG network with an associated edge weight above 52 will be deleted to generate the trimmed graph
 
#### Command for example data
 The command to run step3 on the example data for the Nephrozoa Last Common Ancestor (with a species threshold of 3) is:
```bash
./step3_find_og_commus.py myresults/bynode/Nephrozoa/m_3/myresults.m_3.Nephrozoa.dist  -n 52 -o myresults/Nephrozoa.og_commus
```
this will create two files:
* `Nephrozoa.og_commus.csv`: one OG commu per line. The connected components were isolated from the trimmed graph (here, all edges above 52 were deleted), and "clique-checked". Namely, only sets of vertices that formed cliques of OGs in the untrimmed graph were considered to be ancestral OG communities.
* `Nephrozoa.og_commus.gpickle`: untrimmed graph, saved as a pickled networkx.graph object. Can be read using the NetworkX  python package (using `networkx.read_gpickle("Nephrozoa.og_commus.gpickle")`.

#### Step3 - parameters

*a csv file, one line per og community isolated from the ancestral OG graph. These are not the connected components from the ancestral oggraph trimmed of edges with weights above 52,  

* **node_dist** (positional): node specific dist file created using step2.
* **-n, --nmax**: distance threshold above which syntenic blocks will be considered split. Can be estimated using the `tools/step2.5_optimal_nmax.py` script (default: 30).
* **-o, --output**: prefix of the output. Two files will be created using this prefix, one with `gpickle`, other with `csv` extension.

The OG commus (`csv` file) are ancestral microsyntenic blocks.

### Step4 - recover ancestral microsyntenic blocks in extant genomes
This step recovers ancestral microsyntenic blocks in extant genomes. In an order to limit noise in the case of atomized blocks (all the orthologs are located on different chromosomes), a coverage of at least 30% of the orthogroups found in the ancestral microsyntenic block is required.

**Inputs file:**
* `chromdata.pickle` created by step1
* Pickled ancestral OG network (gpickle format) created by step3
* Og communities created by step3

##### Command for example data
This command will find blocks for the Nephrozoan node, with a species_threshold of 3 for step2, and a nmax of 52 for step3.
```bash
./step4_OG_communities_to_blocks_graph_check.py myresults/Nephrozoa.og_commus.csv -g myresults/Nephrozoa.og_commus.gpickle -c myresults/chromdata.pickle -o myresults/example_blocks
```
This will create two files:
* `example_blocks.len3.ol0.5.clusters`: multi_species blocks, clusters format
* `myresults/example_blocks.len3.ol0.5.synt`: blocks in extant species, in synt format

#### Step4 - Parameters
* **og_communities** (positional): csv of orthogroup communities, generated by step3
* **-g, --filtered_graph**: gpickle output from step3, trimmed using nmax. Paralogs of the same OG will considered as multiple conserved occurences only if there is a self edge in this graph.
* **-l, --min_len**: minimum number of ortholog occurences (One OG counts as a single shared OG occurence, unless there is a self edge in the filtered graph.)  (default: 3).
* **-s, --min_shared**: minimum overlap coefficient of the OG occurences two scaffold should share to be considered homologous (default: .5).
* **-k, --clique_size**: how many links the retained blocks are expected to posess with other blocks (default: 3, i.e. links with two other species)
* **-r, --min_community_coverage**: percentage of OGs of the ancestral syntenic block a given block should posess to be retained (default: 0.3, i.e. 30%)
* **-o, --output**: prefix of the output file

The parameters min_len, clique_size and min_community_coverage are pretty sensible and shouldn't be touched in most cases. But if you want to optimize a run, or limit noise, they could be used (if you already didn't limit noise by using a higher species_threshold in step2).
The way this step works is that a network is build for each ancestral  microsyntenic block, where nodes are chromosomes/scaffolds with a length at least equal to , and edges are drawn only between scaffold of different species sharing `min_len` OG occurences and with an overlap coefficient of at least `min_shared`. Communities are then identified in this network by percolation of cliques of size k (specified through `clique_size` parameter), each community corresponds to a microsyntenic block.
Accordingly,  `min_len` and `min_shared` are the criterias for stating homology of two blocks (we  use the same parameters as defined for the detection  of pairwise syntenic blocks in [Simakov et *al.* 2013](https://doi.org/10.1038/nature11696)). Adjusting these paraneters might limit noise, at the expense of sensitivity. `clique_size` will also influence how homology of blocks is defined, i.e. how many species a block should be found in to be retained in the output. This parameter could be increased if you have a very large taxonomic sample and are interested in  highly conserved blocks. The default is set at 3 (2 homologs required), because monophyly of the syntenic relationships is not a reasonable assumption (see [Winter et *al.*, 2016](https://doi.org/10.1093/nar/gkw843)).

### Final step - check taxonomic composition of the blocks
This step is a final validation step, where one verifies whether the taxonomic composition of the blocks matches with the specified node. For this we use the [BlocksByNode.py](https://github.com/nsmro/comparative_genomics_utils/blob/main/Microsynteny/BlocksByNode.py) script

```bash
BlocksByNode.py -c myresults/example_blocks.len3.ol0.5.clusters -b myresults/example_blocks.len3.ol0.5.synt -s example_data/dataset_test.tre -n Nephrozoa -m 3 -r blocks_list -t total > myresults/example_blocks.len3.ol0.5.taxonomy_filtered.synt
```
In this example, the multi species block 28 (block ids 272,273,274, see `myresults/example_blocks.len3.ol0.5.clusters`) has been filtered our, because found only in BRAFL,PECMA and MIZYE. With a species_threshold of 3, at least 3 deuterostomes and 3 protostomes (or 3 protostomes or 3 deuterostomes and 3 outgroup species) are required to possess the block to be retained. Since the multi species block is found in too few species, it is discarded.