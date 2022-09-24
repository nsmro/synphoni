 ![synphoni_logo](assets/SYNPHONI_logo.png)


# SYNPHONI

SYNPHONI detects sets of genes that were located in close proximity to each other in a given ancestor (ancestral microsyntenic blocks) and their descendants that are still syntenic in extant genomes (syntenic blocks).

Here, the term synteny is meant according to the definition in [Renwick, 1971](https://doi.org/10.1146/annurev.ge.05.120171.000501), i.e. genes located on the same chromosome in two or more species, regardless of conserved gene order (collinearity) or intergenic distance. This is also referred to as conserved macrosynteny (see [Simakov et *al.* 2022](https://doi.org/10.1126/sciadv.abi5884)). Of note, the ancestral microsyntenic blocks identified by SYNPHONI are closer to the definition made in [Pevzner and Tesler 2003](https://doi.org/10.1101/gr.757503), where micro-rearrangements are allowed within syntenic blocks, but macro-rearrangements are not.

All descendants of the ancestral microsyntenic blocks are recovered from all extant genomes where the ancestral synteny has been retained, even if ancestral proximity has not been retained. Furthermore, block sets are inferred for each phylogenetic node of the species sample independently. It is thus possible to reconstruct how a given block evolved across the different lineages of a tree.

In sum, SYNPHONI has been designed for studying the transformation of ancient microsyntenic blocks across all scales of evolutionary time and genome organization (for examples, please see the manuscript associated with SYNPHONI). 

# Getting started

For now, SYNPHONI runs on a single core. A sample of 80 metazoans was processed in less than 48 hours using a single core.

## Requirements

SYNPHONI requires python 3.9 or higher and requires the following python libraries:

* networkx version 2.6.2
* ete3 version 3.1.2
* numpy version 1.21.2
* scipy version 1.7.1

## Inputs and outputs

A SYNPHONI run requires 4 main steps, each specified using the `step.*.py scripts`. In the future, I might create a workflow manager to make it easier to run.

**Before starting**, check that you have all the inputs prepared:

* The positions of the proteins on the genomes (one isoform per gene), *chrom* format including species prefix
* An orthology file where the protein names start with a species prefix (*clus* format)
* A *Newick* tree (branch lengths do not matter only the topology, but may include polytomies) with node labels and species prefixes as tip labels

The output of SYNPHONI, for a given node will be:

* A *synt* file, containing information about the blocks (species, genomi locations, gene content, orthogroups (OGs) of the genes)
* A *clusters* file, displaying which block (of the *synt* file) belongs to which multispecies block 

Descriptions of the **clus**, **chrom**, and **synt/clusters** format, as well as scripts to convert **gff** or **bed** to **chrom** or **Orthofinder** HOGs to **clus** format can be found [Here](https://github.com/nsmro/comparative_genomics_utils/tree/main/Microsynteny)


## Example of a SYNPHONI run

SYNPHONI is composed of 4 steps:

* **Step1**: Build the **dist** file, a matrix with the minimum distances between all syntenic OG pairs that are found in two species or more
* **Step2**: Estimate ancestral distances in specified nodes (ancestral OG graphs), based on the cladogram (**newick** tree) provided by the user
* **Step3**: Find sets of OGs, which correspond to approximations of ancestral microsyntenic blocks in the ancestral OG graphs
* **Step4**:  Recover descendants of the ancestral microsyntenic blocks in extant species (outputs **synt** and **clusters** files)

### Step1 - build a dist file

This step recovers the minimum distances (intevening genes) between any syntenic pair of OGs found in two or more species. All possible OG pair combinations will be examined for all the species. Since the number of genes in metazoans is more or less constant, adding more species should lead to a linear increase of the runtime.

**Input files:**

* orthology file (**clus** format)
* locations of genes (**chrom** format)

#### Command for example data (using the [step1_make_dists.py](https://github.com/nsmro/synphoni/blob/main/step1_make_dists.py) script)

The command to run step1 on the example data is:

```bash
./step1_make_dists.py -c example_data/dataset_test.clus $(ls example_data/chrom_files/*chrom) -o myresults
```

This is the longest step of the pipeline (should take 3-5 minutes on the example data) and creates a folder "myresults" containing two files:

* `myresults.dist`:  the matrix
* `chromdata.pickle`: a pickle file containings the parsed/filtered genome locations (used in step4)

#### Step1 - parameters:

* **-c, --clusfile**: orthology file, in **clus** format
* **-o, --output**: name of the output folder
* **-m, --maxpara** (default: 100): OGs containing more paralogs from a single species than specified will be excluded from all downstream analysis 
* **-s, --species_threshold** (default: 2): minimum number of species in which an OG pair needs to be syntenic to be retained

It's more efficient to leave the species threshold set up at 2 and to filter more stringently in the following steps, especially since step1 has the longest computational time.

### Step2 - infer ancestral OG networks

This step will estimate the ancestral distance between each OG pair (in the **dist** matrix) for each specified phylogenetic node (in the **Newick** cladogram), generating the edgelists of the ancestral OG networks

**Input files:**

* species cladogram (**newick** format, user provided)
* **dist** file (created by step1)


For a given node of interest (node name passed to -n argument), SYNPHONI we definesd the five following taxonomic groups:

* Ingroup = all species descended from a given N
* Sister group = all species of the clade with the closest relationship to the ingroup 
* Outgroup = all species outside of a given ingroup (i.e. including sister group species)
* Ingroup clade = all species of a children clade within a given ingroup
* Outgroup clade = all species of a children clade within a given outgroup 

#### Command for example data (using the [step2_filter_pairs.py](https://github.com/nsmro/synphoni/blob/main/step2_filter_pairs.py) script)

The command to run step2 on the example data for the Nephrozoa Last Common Ancestor is:

```bash
./step2_filter_pairs.py -d myresults/myresults.dist -s example_data/dataset_test.tre -n Nephrozoa
```

This should take up to a minute to process the example dataset, and will create a the `myresults.m_3.Nephrozoa.dist` file in the current working directory (file is named based on the .dist file name, the species_threshold and the node_name). While it is possible to pass multiple node names to a single step2 command, I'd recommendt to limit each step2 command to a single node, and to run the different commands in parallel.

#### Step2 - parameters:

* **-s, --species_tree**: **Newick** tree where leaves are the species prefixes found in the first field of the **chrom** file used in step1
* **-n, --node_names**: space-separated names of the nodes for which you want to build ancestral OGs networks.
* **-m, --species_threshold** (default: 3): minimum number of species of a phylogenetic clade that must possess a syntenic OG pair, for the clade to be considered populated by it (if the threshold is larger than the clade, the syntenic OG pair must be present in all species of the clade to populate it). At least two clades must be populated (two ingroups, one ingroup and at least one sister group, or one ingroup and the outgroup) for an OG pair to be retained as ancestrally syntenic.
* **-d, --dist**: the **dist** file created by step1

The **m** parameter will determine how many edges (OG syntenic relationships) will be included in the ancestral OG network. A high species_threshold (larger than all ingroup clades) means that monophyly of the syntenic relationships is required (not a reasonable assumption, see [Winter et *al.*, 2016](https://doi.org/10.1093/nar/gkw843)). A species_threshold of one, on the other hand, will lead to the detection of a lot of noise (e.g. convergent rearrangements, misassigned orthologs).

### Step3 - Isolate OG sets from the ancestral OG networks

This step will isolate ancestrally microsyntenic OG blocks for every given node. It relies on a dynamically determined distance threshold `nmax` to isolate subgraphs from the ancestral OG network. All edges with an edge weight (ancestral intergenic distance) above this estimated `nmax` will be deleted from the graph, leaving only connected components (which are further refined, see associated manuscript for details).

**Input files:**

* node-specific **dist** file generated by with step 2

#### Finding nmax automatically with the [step2.5_optimal_nmax.py](https://github.com/nsmro/synphoni/tree/main/tools/step2.5_optimal_nmax.py) script

There is a script to estimate nmax provided in tools, but it relies on a folder structure (node_name/m_*) to determine the node names and species_thresholds used for the node-specific **dist** files.


##### Step2.5 parameters

* **positional argument**: the path to the parent of the nested folder structure (here "bynode")
* **-o, --output**: desired prefix of the output files

The following three commands will create a nested folder structure within a folder called `bynode`, move the **dist** file into it, and run the script to estimate `nmax`:

```bash
mkdir -p myresults/bynode/Nephrozoa/m_3 # creates the directory and its parents
mv myresults.m_3.Nephrozoa.dist  myresults/bynode/Nephrozoa/m_3 # move dist file into the folder (here, only one node and m parameter)
tools/step2.5_optimal_nmax.py myresults/bynode/ -o myresults/nmax_test
```

Two files are created with this command (within the "myresults" folder):

* `nmax_test.tsv.component.csv`: table (csv format) comprising the size of the largest connected component with varying nmax thresholds, for all the nodes and associated species_threshold parameters found in the "bynode" folder. Columns are as follows:
    * **node**: name of the node (gathered from the folder names)
    * **m**: *species_threshold* of step2 used to generate the ancestral OG graph (gathered from the folder names)
    * **nmax**: all the values nmax took (*start*, *stop*, *step_size* are three space separated arguments passed to the `-n, --nmax option`. Defaults are 1, 100, and 1, i.e. all the values between 1 and 100 with increments of 1)
    * **proportion**: proportion of all the vertices (OGs) of the ancestral OG  network/graph that are part of the largest connected component
* `nmax_test.tsv.inflection.csv`: table (csv format) comprising the suggested nmax (x-left column) for each node/species_threshold pair. Columns of the table are as follows:
    * **node**: name of the node (gathered from the folder names)
    * **m**: *species_threshold* used to generate the ancestral OG graph (gathered from the folder names)
    * **x-left**: estimated optimal *nmax*. The size of the largest component as a function of the largest *nmax* follows a sigmoid curve (increasing curve, i.e. convex portion, then concave portion). The estimated *nmax* is the elbow of the convex portion of the curve (see the associated manuscript for details).

 
In the example data, if `node=Nephrozoa` and `species_threshold m=3`, the estimated nmax is **52**. This means that all the edges of the ancestral OG network with an associated edge weight above 52 will be deleted to generate the trimmed graph

 
#### Command for example data using the [step3_find_og_commus.py](https://github.com/nsmro/synphoni/blob/main/step3_find_og_commus.py) script)

The command to run step3 on the example data for the Nephrozoa Last Common Ancestor (with a species threshold of 3) is:

```bash
./step3_find_og_commus.py myresults/bynode/Nephrozoa/m_3/myresults.m_3.Nephrozoa.dist  -n 52 -o myresults/Nephrozoa.og_commus
```

this will create two files:

* `Nephrozoa.og_commus.csv`: table with one OG community per line. The connected components were isolated from the trimmed graph (here, all edges above 52 were deleted), and "clique-checked" (i.e. only sets of vertices/OGs that also formed cliques in the untrimmed graph were retained as ancestral OG communities)
* `Nephrozoa.og_commus.gpickle`: untrimmed graph, saved as a pickled networkx.graph object. Can be read using the NetworkX python package (using `networkx.read_gpickle("Nephrozoa.og_commus.gpickle")`


#### Step3 - parameters

* **node_dist** (positional): node specific **dist** file created using step2.
* **-n, --nmax** (default: 30): distance threshold above which syntenic blocks will be considered split. Can (and should) be estimated using the `tools/step2.5_optimal_nmax.py` script.
* **-o, --output**: prefix of the output. Two files will be created using this prefix, one with `gpickle`, other with `csv` extension.

The obtained OG communities (`csv` file) are ancestral microsyntenic blocks.

### Step4 - recover ancestral microsyntenic blocks in extant genomes

This step recovers descendants of ancestral microsyntenic blocks from extant genomes. In order to limit noise produced by atomized blocks (i.e. when most orthologs have translocated to different chromosomes), a minimum coverage (`-r` = 0.3) of the ancestral microsyntenic block is required to retain a descendant syntenic block. This means that at least 30% of the OGs of an ancestral microsyntenic block must “co-occur” in an extant block. An obtained extant species block thus corresponds to a DNA strand/scaffold bearing a minimum number of the members of an ancestral microsyntenic block. To confirm that synteny is conserved among extant species, we build another network, where extant blocks are vertices and while edges indicate the presence of a sufficient number of OG co-occurrences between two blocks from different species (`-l` = 3 and `-s` = 0.5). Communities/multispecies blocks are then identified in this network by percolation of cliques of size `-k` (= minimum number of species where the block must be conserved). 

**Input files:**

* `chromdata.pickle` created by step1
* Ancestral OG network created by step3 (in this example, `Nephrozoa.og_commus.gpickle`)
* OG communities created by step3 (in this example `Nephrozoa.og_commus.csv`)

##### Command for example data using the [step4_OG_communities_to_blocks_graph_check.py](https://github.com/nsmro/synphoni/blob/main/step4_OG_communities_to_blocks_graph_check.py) script)

This command will find blocks for the Nephrozoan node, with a *species_threshold* m of 3 (from step2), and a *nmax* of 52 (from step3).

```bash
./step4_OG_communities_to_blocks_graph_check.py myresults/Nephrozoa.og_commus.csv -g myresults/Nephrozoa.og_commus.gpickle -c myresults/chromdata.pickle -o myresults/example_blocks
```

This will create two files:

* `example_blocks.len3.ol0.5.clusters`: multispecies blocks, **clusters** format
* `myresults/example_blocks.len3.ol0.5.synt`: blocks in extant species, in **synt** format

#### Step4 - parameters

* **og_communities** (positional): **csv** of the OG communities generated by step3
* **-g, --filtered_graph**: **gpickle** output from step3 (OG network trimmed using nmax), note that paralogs of the same OG will only be considered as multiple conserved co-occurrences if they have a  self edge (see associated manuscript for details)-
* **-l, --min_len** (default: 3): minimum number of OG co-occurrences between two extant species blocks (each shared OG counts as a single OG co-occurrence, unless it has  a self edge in the ancestral network)
* **-s, --min_shared** (default: 0.5): minimum overlap coefficient between the OG occurrences of two extant species blocks (scaffolds)
* **-k, --clique_size** (default: 3): minimum number of edges per community/multispecies block (if k = 3 then a multispecies block is only retained if it is conserved across at least three species)
* **-r, --min_community_coverage** (default: 0.3, i.e. 30%): percentage of OGs of the ancestral syntenic block that an extant species block should possess to be retained
* **-o, --output**: prefix of the output file

The parameters `min_len` (**-l**), `clique_size` (**-k**) and `min_community_coverage` (**-r**) are pretty sensible and shouldn't be touched in most cases. However, if you want to optimize a run or limit noise, they could be used (e.g. if you didn't already limit noise by using a higher `species_threshold` **m** in step2).  The min_len (**-l**) and min_shared (**-s**) parameters are the criterias for stating homology of two extant blocks (we use the same parameters as defined for the detection of pairwise syntenic blocks in [Simakov et *al.* 2013](https://doi.org/10.1038/nature11696)). Increasing these parameters can  reduce noise, but at the expense of sensitivity. The `clique_size` (**-k**) determines how widely a block must be conserved across extant species  to be retained in the output. Therefore, this parameter could be increased, if you have a very large taxonomic sample and are only interested in highly conserved blocks. The default is set at 3 (i.e. at least 3 homologous blocks required), because monophyly of syntenic relationships is not a reasonable assumption (see [Winter et *al.*, 2016](https://doi.org/10.1093/nar/gkw843)).

### Final step - check taxonomic composition of the blocks

This is a final validation step that verifies whether the taxonomic composition/coverage of each obtained multispecies block satisfies the requirements (see associated manuscript) of the specified node. 

**Input files:**

* **example_blocks.len3.ol0.5.clusters**: multispecies blocks outputted by step4
* **myresults/example_blocks.len3.ol0.5.synt**: species blocks outputted by step4
* **example_data/dataset_test.tre**: **Newick** cladogram used in step2

##### Command for example data using the [BlocksByNode.py](https://github.com/nsmro/comparative_genomics_utils/blob/main/Microsynteny/BlocksByNode.py) script

```bash
BlocksByNode.py -c myresults/example_blocks.len3.ol0.5.clusters -b myresults/example_blocks.len3.ol0.5.synt -s example_data/dataset_test.tre -n Nephrozoa -m 3 -r blocks_list -t total > myresults/example_blocks.len3.ol0.5.taxonomy_filtered.synt
```

This command creates a new **synt** file containing the validated species blocks. 

```bash
BlocksByNode.py -c myresults/example_blocks.len3.ol0.5.clusters -b myresults/example_blocks.len3.ol0.5.synt -s example_data/dataset_test.tre -n Nephrozoa -m 3 -r clusters_list -t total | cut -f 12,4- > myresults/example_blocks.len3.ol0.5.taxonomy_filtered.clusters
```

This command creates a new **clusters** file containing the validated multi-species blocks. 


In this example, for the Nephrozoa node, the multispecies block 28 (includes species block IDs 272, 273 and 274, see `myresults/example_blocks.len3.ol0.5.clusters`) is discarded, because it is found only in BRAFL, PECMA and MIZYE. With a species_threshold of m = 3, at least 3 species per ingroup clade (i.e. 3 deuterostomes and 3 protostomes) or 3 species from one ingroup clade and 3 species from the outgroup are required to justify block retention

#### Final step - parameters

* **-c, --clusters** : multispecies blocks outputted by step4 (**clusters** format) 
* **-b, --block_list** : species blocks outputted by step4 (**synt** format)
* **-s, __species_tree** : **Newick** cladogram (.tre format)
* **-n, --node_name** : name of the node for which you want to validate the (multi-)species blocks outputted by step4
* **-m, --species_threshold** : see step2
* **-r, --report** : output type (e.g. blocks_list = list of species blocks, clusters_list = list of multispecies blocks) 


### Optional step - count intervening genes

This step determines the number of intervening genes between consecutive members of all species blocks by node. 

**Input files:**

* **myresults/chromdata.pickle**: pickle file containing the parsed/filtered genome locations created in step1
* **myresults/example_blocks.len3.ol0.5.taxonomy_filtered.synt**: validated species blocks outputted by the final validation step
* **myresults/example_blocks.len3.ol0.5.clusters**: multispecies blocks outputted by step4

##### Command for example data using the [analysis_intervening_genes.py](httpshttps://github.com/nsmro/synphoni/blob/main/analysis_intervening_genes.py) script

```bash
./analysis_intervening_genes.py -c myresults/chromdata.pickle -sy myresults/example_blocks.len3.ol0.5.taxonomy_filtered.synt -ms myresults/example_blocks.len3.ol0.5.clusters -o myresults/example_blocks.intervening
```

This command creates a table (.tsv format) where each line contains the number of intervening genes between two members of a (multi-)species block. The seven columns are as follows: 
	* Multispecies block ID (corresponds to **clusters** files)
	* Species block ID (corresponds to **clusters** and **synt** files)
	* Chromosome ID
	* Species prefix
	* Comma separated protein accessions of the two block members
	* Paralog vs. ortholog information (“para” if the two block members are paralogs or “not_para” if they are not)
	* Number of intervening genes

#### Step4 - parameters

* **-c, --chrom_data** : pickle file containings the parsed/filtered genome locations created in step1
* **-sy. --synt_file** : filtered species blocks outputted by the final validation step (**synt** format)
* **-ms, --multi_sp_file** : multispecies blocks outputted by step4 (**clusters** format)
* **-o, –-output**: prefix of the output **tsv** file