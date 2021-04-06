# OpenKinome

<!-- Written by Jaime Rodríguez-Guerra, Apr 2021 -->

The [OpenKinome](https://github.com/openkinome) initiative aims to leverage the increasingly available bioactivity data and scalable computational resources to perform kinase-centric drug design in the context of structure-informed machine learning and free energy calculations.

## The goals

Kinases are a well studied group of proteins due to their role in cancer and other diseases. As a result, there is an abundance of data waiting to be used in computational exercises such as machine learning.

The hypothesis is that using higher quality data will allow us to build models with improved prediction accuracy. Eventually, we aim to build structure-informed models that can benefit from learn from free energy calculations in some kind of reinforcement learning scheme.

However, to get there we must establish baseline models and a way to iterate and improve in reproducible and comparable ways. This calls for the following pillars:

- **Robust and extensible object models**: all data must be represented in the same way, so it can be compared and reused.
- **Guaranteed provenance**: every execution is logged and every input data is versioned and archived. If an experiment needs to be debugged, it can be run again, no matter where or when or by whom. This leads to:
- **Reproducible pipelines, down to the bit**: automated workflows will deterministically recreate the same models given the same input configuration data.

## The challenges and the requirements

Getting the available data to work reliably in our machine learning studies requires some prior work. Some of the key challenges we need to solve to get to that point are summarized below.

### Obtaining raw data

The origin of the data is very important for reproducibility. The source and uniquely identifying tags need to be annotated and kept. Ideally, this means that you are using versioned data (a git tag, a DOI, etc).

For example, Drewry's Published Kinase Inhibitor Set 2 (PKIS2) dataset can be [downloaded from the Supporting Information](https://journals.plos.org/plosone/article?id=10.1371/journal.pone.0181585#sec015) attached to the corresponding [manuscript](https://journals.plos.org/plosone/article?id=10.1371/journal.pone.0181585). Since it has DOI, it can be traced back to its source.

Online webservices such as [ChEMBL](https://www.ebi.ac.uk/chembl/) contain a lot of valuable data, sprinkled with sometimes less useful data. Retrieving the useful bits can be automated with packages such as [`chembl_webresource_client`](https://github.com/chembl/chembl_webresource_client) but it does not provide a way to version the acquisition (other than the current date, which makes it not reproducible). Fortunately, ChEMBL does offer [archived database dumps](https://chembl.gitbook.io/chembl-interface-documentation/downloads#chembl-database-release-dois) you can download and query locally.

The repository [`openkinome/kinodata`](https://github.com/openkinome/kinodata) governs the process of querying (and cleaning) wide-purpose databases such as ChEMBL for the parts we need for our experiments. If you need to obtain data from an online resource and guarantee its provenance and reproducibility, this is the place you add your contributions. If new querying and curation protocols need to be added to the pipeline, [a new release will be cut](https://github.com/openkinome/kinodata/releases/tag/v0.1) and the the resulting CSV files will be versioned and archived there for posterity.

### From raw data to a unified object model

One of the most useful types of data we can find in the context of drug design is _binding affinity_ or `ΔG`, which involves at least three elements: the measurement itself (a number), and two molecular counterparts (a protein or _target_, and a small compound or _ligand_). The relationship between measurements and molecules is a bit more nuanced than you think:

- The same ligand can be measured against several proteins.
- The same protein can be targetted with different ligands.
- A protein-ligand combination can be called a system or complex, and it can be measured once, several times under different conditions (different experiments), or even several times under the same conditions (replicates).

A the same time, data itself can come from very diverse sources, formatted in different ways. Excel spreadsheets, CSV files, SQL databases, HTML tables, web applications... No matter the source, we need to funnel that data into a **unified object model** that is able to represent the observed measurements in a flexible yet concise way. If we can aim for a sufficiently robust object model, we can automate any data conversions along the way. This process is governed by `kinoml.datasets`. The `kinoml.core` subpackage specifies the following object model:

- A `DatasetProvider` is essentially a list of `Measurement` objects.
- A `Measurement` is essentially a float (singlicate) or a list of floats (replicates), associated to a `System` plus some experimental conditions.
- A `System` is a list of `MolecularComponent` objects; usually a `Protein` and a `Ligand`.
- `MolecularComponent` objects can be of very different nature, depending on how the raw data for that molecular entity is encoded.

### Molecular components, representations and entities

Published datasets are not usually great when it comes to providing information about what was measured _exactly_. They will contain enough data to identify the intent, but maybe not enough to reconstruct the experimental conditions in a computational workflow. For example:

Information about the measured proteins might be encoded via:

- Their corresponding gene names (what about mutations? PTMs? splicing variants?)
- Their UniProt identifier
- Their FASTA sequence
- A PDB identifier.

Ligands might be encoded with:

- IUPAC names
- (Non-canonicalized) SMILES
- Inchi keys
- SDF files...

In the end, it should not matter, because they all represent the same molecular _entity_, and the object model must be able to encode for that. We strive to provide an object model that _can_ funnel all these differents molecular representations into the same idealized representation of the molecular entity. Notice we mention the _possibility_, not the _requirement_. Some workflows do not need to go through the expensive demands of constructing a fully-fledged `Protein` object out of a PDB identifier, and hence, they can escape that data augmentation if the dataset contains _enough_ information for the needs of the experiment. That does not mean that the workflow is not reproducible, because in the end a part of the object model tree was used to annotate the objects with sufficient provenance information.

[More details](http://openkinome.org/kinoml/api/core/components/).

### Measurement types and uncertainties

Most experimental assays do not estimate ΔG of binding directly, but through proxy measurements like `IC50`, `Kd`, `Ki`, or percentage of displacement. Additionally, those assays are performed by different laboratories with different protocols and techniques. This all leads to different experimental uncertainties and _soft_ measurements. While the data can be a bit noisy, it is definitely informative and has the potential to provide predictive power in a adequately tuned model.

This results in the following requirements:

- The framework must handle different **measurement types** that, nonetheless, represent the same underlying physical phenomenon: ΔG.
- To leverage several measurement types in the same learning exercise, the models will be instructed to _learn_ ΔG. Since the losses will need to be computed against the observed measurement (e.g. IC50), this calls for mathematical expressions that can convert between both. As such, each measurement type will feature its own **observation model**.
- For the sake of reproducibility, each measurement needs to be annotated with **provenance** data (source dataset, publication...).
- To combine different sources of data, measurements need to contain information about the **uncertainty**. When this is not available, it can be estimated or learned.

[More details](http://openkinome.org/kinoml/api/core/measurements/).

## The software

Achieving all these goals and fulfilling all these requirements require modular libraries that allow us to compose the needed pipelines.

Our main library is [`openkinome/kinoml`](https://github.com/openkinome/kinoml/). It provides all the building blocks at the different stages of the pipeline. Some highlights:

- Data ingestion is done at `kinoml.datasets`.
- The unified object model is specified at `kinoml.core`.
- The different elements of the featurization pipelines can be found at `kinoml.features`.
- Machine learning models are provided in `kinoml.ml`. This subpackage also provides tooling to juggle different ML backends (PyTorch, XGBoost...).

The result-oriented pipelines are built in additional repositories, prefixed with `experiments-*`. For example, [`openkinome/experiments-binding-affinity`](https://github.com/openkinome/experiments-binding-affinity) provides an automated notebook running system based on templates and `papermill`. It encourages separating featurization from training by design, so the same tensors can be reused across different models; also, the same models can be trained with different featurization schemes.

Finally, providing curated data ready to be ingested at `kinoml` is tackled by [`openkinome/kinodata`](https://github.com/openkinome/kinodata). Here, several notebooks query different online resources to obtain an updated list of the human kinome and the presence of relevant bioactivity in different datasets, like ChEMBL. CSV artifacts are released on GitHub and downloaded and cached by `kinoml.datasets` during featurization.

## The team

The OpenKinome initiative is the product of an ongoing collaboration between [Volkamer lab](https://volkamerlab.org/) (Berlin, DE) and [Chodera lab](https://www.choderalab.org/) (New York, US).

## Acknowledgements

Funded by [Stiftung Charité](https://www.stiftung-charite.de/) ([Einstein BIH Visiting Fellow Project](https://www.einsteinfoundation.de/en/people-projects/einstein-bih-visiting-fellows/john-chodera/)), Bayer AG, and others.
