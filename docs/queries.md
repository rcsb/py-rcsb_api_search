# Queries

Two syntaxes are available for constructing queries: an "operator" API using python's
comparators, and a "fluent" API where terms are chained together. Which to use is a
matter of preference, and both construct the same query object.

## Operator syntax

Searches are built up from a series of `Terminal` nodes, which compare structural
attributes to some search value. In the operator syntax, python's comparator
operators are used to construct the comparison. The operators are overloaded to
return `Terminal` objects for the comparisons.
```python
from rcsbsearchapi.search import TextQuery
from rcsbsearchapi import rcsb_attributes as attrs

# Create terminals for each query
q1 = TextQuery("heat-shock transcription factor")
q2 = attrs.rcsb_struct_symmetry.symbol == "C2"
q3 = attrs.rcsb_struct_symmetry.kind == "Global Symmetry"
q4 = attrs.rcsb_entry_info.polymer_entity_count_DNA >= 1
```
Attributes are available from the rcsb_attributes object and can be tab-completed.
They can additionally be constructed from strings using the `Attr(attribute)`
constructor. For a full list of attributes, please refer to the [RCSB
schema](http://search.rcsb.org/rcsbsearch/v2/metadata/schema).

`Terminal`s are combined into `Group`s using python's bitwise operators. This is
analogous to how bitwise operators act on python `set` objects. The operators are
lazy and won't perform the search until the query is executed.
```python
query = q1 & (q2 & q3 & q4)  # AND of all queries
```
AND (`&`), OR (`|`), and terminal negation (`~`) are implemented directly by the API,
but the python package also implements set difference (`-`), symmetric difference (`^`),
and general negation by transforming the query.

Queries are executed by calling them as functions. They return an iterator of result
identifiers.
```python
results = set(query())
```
By default, the query will return "entry" results (PDB IDs). It is also possible to
query other types of results (see [return-types](http://search.rcsb.org/#return-type)
for options):
```python
assemblies = set(query("assembly"))
```
### Fluent syntax

The operator syntax is great for simple queries, but requires parentheses or
temporary variables for complex nested queries. In these cases the fluent syntax may
be clearer. Queries are built up by appending operations sequentially.
```python
from rcsbsearchapi.search import TextQuery, AttributeQuery, Attr

# Start with a Attr or TextQuery, then add terms
results = TextQuery("heat-shock transcription factor").and_(
    # Add attribute node as fully-formed AttributeQuery
    AttributeQuery("rcsb_struct_symmetry.symbol", "exact_match", "C2") \
    # Add attribute node as Attr with chained operations
    .and_(Attr("rcsb_struct_symmetry.kind")).exact_match("Global Symmetry") \
    # Add attribute node by name (converted to Attr) with chained operations
    .and_("rcsb_entry_info.polymer_entity_count_DNA").greater_or_equal(1)
    ).exec("assembly")

# Exec produces an iterator of IDs
for assemblyid in results:
    print(assemblyid)
```
### Structural Attribute Search and Chemical Attribute Search Combination

Grouping of a Structural Attribute query and Chemical Attribute query is permitted as long as grouping is done correctly and search services are specified accordingly. Not the example below. More details on attributes that are available for text searches can be found on the [RCSB PDB Search API](https://search.rcsb.org/#search-attributes) page.
```python
from rcsbsearchapi.const import CHEMICAL_ATTRIBUTE_SEARCH_SERVICE, STRUCTURE_ATTRIBUTE_SEARCH_SERVICE
from rcsbsearchapi.search import AttributeQuery

# By default, service is set to "text" for structural attribute search
q1 = AttributeQuery("exptl.method", "exact_match", "electron microscopy",
                    STRUCTURE_ATTRIBUTE_SEARCH_SERVICE # this constant specifies "text" service
                    )

# Need to specify chemical attribute search service - "text_chem"
q2 = AttributeQuery("drugbank_info.brand_names", "contains_phrase", "tylenol",
                    CHEMICAL_ATTRIBUTE_SEARCH_SERVICE # this constant specifies "text_chem" service
                    )

query = q1 & q2 # combining queries

list(query())
```
### Computed Structure Models

The [RCSB PDB Search API](https://search.rcsb.org/#results_content_type)
page provides information on how to include Computed Models into a search query. Here is a code example below.
This query returns ID's for experimental and computed models associated with "hemoglobin". 
Queries with only computed models or only experimental models can be made.
```python
from rcsbsearchapi.search import TextQuery

q1 = TextQuery("hemoglobin")

# add parameter as a list with either "computational" or "experimental" or both as list values
q2 = q1(return_content_type=["computational", "experimental"])

list(q2)
```
### Return Types and Attribute Search

A search query can return different result types when a return type is specified. 
Below are examples on specifying return types Polymer Entities,
Non-polymer Entities, Polymer Instances, and Molecular Definitions, using a Structure Attribute query. 
More information on return types can be found in the 
[RCSB PDB Search API](https://search.rcsb.org/#building-search-request) page.
```python
from rcsbsearchapi.search import AttributeQuery

q1 = AttributeQuery("rcsb_entry_container_identifiers.entry_id", "in", ["4HHB"]) # query for 4HHB deoxyhemoglobin

# Polymer entities
for poly in q1("polymer_entity"): # include return type as a string parameter for query object
    print(poly)
    
# Non-polymer entities
for nonPoly in q1("non_polymer_entity"):
    print(nonPoly)
    
# Polymer instances
for polyInst in q1("polymer_instance"):
    print(polyInst)
    
# Molecular definitions
for mol in q1("mol_definition"):
    print(mol)
```
### Protein Sequence Search Example

Below is an example from the [RCSB PDB Search API](https://search.rcsb.org/#search-example-3) page, 
using the sequence search function.
This query finds macromolecular PDB entities that share 90% sequence identity with
GTPase HRas protein from *Gallus gallus* (*Chicken*).
```python
from rcsbsearchapi.search import SequenceQuery

# Use SequenceQuery class and add parameters
results = SequenceQuery("MTEYKLVVVGAGGVGKSALTIQLIQNHFVDEYDPTIEDSYRKQVVIDGET" +
                        "CLLDILDTAGQEEYSAMRDQYMRTGEGFLCVFAINNTKSFEDIHQYREQI" +
                        "KRVKDSDDVPMVLVGNKCDLPARTVETRQAQDLARSYGIPYIETSAKTRQ" +
                        "GVEDAFYTLVREIRQHKLRKLNPPDESGPGCMNCKCVIS", 1, 0.9)
    
# results("polymer_entity") produces an iterator of IDs with return type - polymer entities
for polyid in results("polymer_entity"):
    print(polyid)
```
### Sequence Motif Search Example

Below is an example from the [RCSB PDB Search API](https://search.rcsb.org/#search-example-6) page,
using the sequence motif search function. 
This query retrives occurences of the His2/Cys2 Zinc Finger DNA-binding domain as
represented by its PROSITE signature. 
```python
from rcsbsearchapi.search import SeqMotifQuery

# Use SeqMotifQuery class and add parameters
results = SeqMotifQuery("C-x(2,4)-C-x(3)-[LIVMFYWC]-x(8)-H-x(3,5)-H.",
                        pattern_type="prosite",
                        sequence_type="protein")

# results("polymer_entity") produces an iterator of IDs with return type - polymer entities
for polyid in results("polymer_entity"):
    print(polyid)
```

You can also use a regular expression (RegEx) to make a sequence motif search.
As an example, here is a query for the zinc finger motif that binds Zn in a DNA-binding domain:
```python
from rcsbsearchapi.search import SeqMotifQuery

results = SeqMotifQuery("C.{2,4}C.{12}H.{3,5}H", pattern_type="regex", sequence_type="protein")

for polyid in results("polymer_entity"):
    print(polyid)
```

You can use a standard amino acid sequence to make a sequence motif search. 
X can be used to allow any amino acid in that position. 
As an example, here is a query for SH3 domains:
```python
from rcsbsearchapi.search import SeqMotifQuery

# By default, the pattern_type argument is "simple" and the sequence_type argument is "protein".
results = SeqMotifQuery("XPPXP")  # X is used as a "variable residue" and can be any amino acid. 

for polyid in results("polymer_entity"):
    print(polyid)
```

All 3 of these pattern types can be used to search for DNA and RNA sequences as well.
Demonstrated are 2 queries, one DNA and one RNA, using the simple pattern type:
```python
from rcsbsearchapi.search import SeqMotifQuery

# DNA query: this is a query for a T-Box.
dna = SeqMotifQuery("TCACACCT", sequence_type="dna")

print("DNA results:")
for polyid in dna("polymer_entity"):
    print(polyid)

# RNA query: 6C RNA motif
rna = SeqMotifQuery("CCCCCC", sequence_type="rna")
print("RNA results:")
for polyid in rna("polymer_entity"):
    print(polyid)
```
### Structure Similarity Query Example

The PDB archive can be queried using the 3D shape of a protein structure. To perform this query, 3D protein structure data must be provided as an input or parameter, A chain ID or assembly ID must be specified, whether the input structure data should be compared to Assemblies or Polymer Entity Instance (Chains) is required, and defining the search type as either strict or relaxed is required. More information on how Structure Similarity Queries work can be found on the [RCSB PDB Structure Similarity Search](https://www.rcsb.org/docs/search-and-browse/advanced-search/structure-similarity-search) page.
```python
from rcsbsearchapi.search import StructSimilarityQuery

# Basic query: querying using entry ID and default values assembly ID "1", operator "strict", and target search space "Assemblies"
q1 = StructSimilarityQuery(entry_id="4HHB")

# Same example but with parameters explicitly specified
q1 = StructSimilarityQuery(structure_search_type="entry_id",
                           entry_id="4HHB",
                           structure_input_type="assembly_id",
                           assembly_id="1",
                           operator="strict_shape_match",
                           target_search_space="assembly"
                           )
for id in q1("assembly"):
    print(id)
```
Below is a more complex example that utilizes chain ID, relaxed search operator, and polymer entity instance or target search space. Specifying whether the input structure
type is chain id or assembly id is very important. For example, specifying chain ID as the input structure type but inputting an assembly ID can lead to
an error.
```python
from rcsbsearchapi.search import StructSimilarityQuery

# More complex query with entry ID value "4HHB", chain ID "B", operator "relaxed", and target search space "Chains"
q2 = StructSimilarityQuery(structure_search_type="entry_id",
                                   entry_id="4HHB",
                                   structure_input_type="chain_id",
                                   chain_id="B",
                                   operator="relaxed_shape_match",
                                   target_search_space="polymer_entity_instance")
list(q2())
```
Structure similarity queries also allow users to upload a file from their local computer or input a file url from the website to query the PDB archive for similar proteins. The file represents a target protein structure in the file formats "cif", "bcif", "pdb", "cif.gz", or "pdb.gz". If a user wants to use a file url for queries, the user must specify the structure search type, the value (being the url), and the file format of the file. This is also the same case for file upload, except the value is the absolute path leading to the file that is in the local machine. An example for file url is below for 4HHB (hemoglobin).
```python
from rcsbsearchapi.search import StructSimilarityQuery

q3 = StructSimilarityQuery(structure_search_type="file_url",
                           file_url="https://files.rcsb.org/view/4HHB.cif",
                           file_format="cif")
list(q3())

# If you want to upload your own structure file for similarity search, you can do so by using the `file_path` parameter:
q4 = StructSimilarityQuery(structure_search_type="file_upload",
                           file_path="/path/to/file.cif",  # specify local model file path
                           file_format="cif")
list(q4())
```

### Structure Motif Query Example

The PDB Archive can also be queried by using a "motif" found in these 3D structures. To perform this type of query, an entry_id or a file URL/path must be provided, along with residues (which are parts of 3D structures.) This is the bare minimum needed to make a search, but there are lots of other parameters that can be added to a Structure Motif Query. 

To make a Structure Motif Query, you must first define anywhere from 2-10 "residues" that will be used in the query. Each individual residue has a Chain ID, Operator, Residue Number, and Exchanges (optional) that must be delcared in that order, or using positional arguments. 

Each residue can only have a maximum of 4 Exchanges, and each query can only have 16 exchanges total. Violating any of these rules will cause the package to throw an Assertion Error. 

Examples of how to instantiate Residues can be found below. These can then be put into a list and passed through to a Structure Motif Query.
```python
from rcsbsearchapi.search import StructureMotifResidue

# construct a Residue with a Chain ID of A, an operator of 1, a residue 
# number of 192, and Exchanges of "LYS" and "HIS"
Res1 = StructureMotifResidue("A", "1", 192, ["LYS", "HIS"])
# as for what is a valid "Exchange", the package provides these as a literal,
# and they should be type checked. 

# you can also use positional arguments:
# this query is the same as above. 
Res2 = StructureMotifResidue(struct_oper_id="1", label_asym_id="A", exchanges=["LYS", "HIS"], label_seq_id=192)

# after delcaring a minimum of 2 and as many as 10, they can be passed into a list for use in the query itself:
Res3 = StructureMotifResidue("A", "1", 162)  # exchanges are optional

ResList = [Res1, Res3]
```
From there, these Residues can be used in a query. As stated before, you can only include 2 - 10 residues in a query. If you fail to provide residues for a query, or provide the wrong amount, the package will throw a ValueError. 

For a Structure Motif Query using an entry_id, the only other necessary value that must be passed into the query is the residue list. The default type of query is an entry_id query. 

As this type of query has a lot of optional parameters, do *not* use positional arguments as more than likely an error will occur. 

Below is an example of a basic entry_id Structure Motif Query, with the residues declared earlier:
```python
from rcsbsearchapi.search import StructMotifQuery

q1 = StructMotifQuery(entry_id="2MNR", residue_ids=ResList)
list(q1())
```
Like with Structure Similarity Queries, a file url or filepath can also be provided to the program. These can take the place of an entry_id. 

For a file url query, you *must* provide both a valid file URL (a string), and the files file extension (also as a string). Failure to provide these elements correctly will cause the package to throw an AssertionError. 

Below is an example of the same query as above, only this time providing a file url:
```python
link = "https://files.rcsb.org/view/2MNR.cif"
q2 = StructMotifQuery(querytype="file_url", url=link, file_extension="cif", residue_ids=ResList)
# querytype MUST be provided. A mismatched query type will cause an error. 
list(q2())
```
Like with Structure Similarity Queries, a filepath to a file may also be provided. This file must be a valid file accepted by the search API. A file extension must also be provided with the file upload. 

The query would look something like this. Note that this is abstracted for the purpose of notebook portability. 

```python
filepath = "/absolute/path/to/file.cif"
q3 = StructMotifQuery(querytype="file_upload", filepath=filepath, file_extension="cif", residue_ids=ResList)

list(q3())
```
There are many additional parameters that Structure Motif Query supports. These include a variety of features such as backbone distance tolerance, side chain distance tolerance, angle tolerance, RMSD cutoff, limits(stop searching after this many hits), atom pairing schemes, motif pruning strategy, allowed structures, and excluded structures. These can be mixed and matched as needed to make accurate and useful queries. All of these have some default value which is used when a parameter isn't provided. These parameters conform to the defaults used by the Search API. 

Below will demonstrate how to define these parameters using non-positional arguments: 
```python
# specifying backbone distance tolerance: 0-3, default is 1
# allowed backbone distance tolerance in Angstrom. 
backbone = StructMotifQuery(entry_id="2MNR", backbone_distance_tolerance=2, residue_ids=ResList)
list(backbone())

# specifying sidechain distance tolerance: 0-3, default is 1
# allowed side-chain distance tolerance in Angstrom.
sidechain = StructMotifQuery(entry_id="2MNR", side_chain_distance_tolerance=2, residue_ids=ResList)
list(sidechain())

# specifying angle tolerance: 0-3, default is 1
# allowed angle tolerance in multiples of 20 degrees. 
angle = StructMotifQuery(entry_id="2MNR", angle_tolerance=2, residue_ids=ResList)
list(angle())

# specifying RMSD cutoff: >=0, default is 2
# Threshold above which hits will be filtered by RMSD
rmsd = StructMotifQuery(entry_id="2MNR", rmsd_cutoff=1, residue_ids=ResList)
list(rmsd())

# specifying limit: >=0, default excluded
# Stop accepting results after this many hits. 
limit = StructMotifQuery(entry_id="2MNR", limit=100, residue_ids=ResList)
list(limit())

# specifying atom pairing scheme, default = "SIDE_CHAIN"
# ENUM: "ALL", "BACKBONE", "SIDE_CHAIN", "PSUEDO_ATOMS"
# this is typechecked by a literal. 
# Which atoms to consider to compute RMSD scores and transformations. 
atom = StructMotifQuery(entry_id="2MNR", atom_pairing_scheme="ALL", residue_ids=ResList)
list(atom())

# specifying motif pruning strategy, default = "KRUSKAL"
# ENUM: "NONE", "KRUSKAL"
# this is typechecked by a literal in the package. 
# Specifies how many query motifs are "pruned". KRUSKAL leads to less stringent queries, and faster results.
pruning = StructMotifQuery(entry_id="2MNR", motif_pruning_strategy="NONE", residue_ids=ResList)
list(pruning())
```
The Structure Motif Query can be used to make some very specific queries. Below is an example of a query that retrives occurances of the enolase superfamily, a group of proteins diverse in sequence and structure that are all capable of abstracting a proton from a carboxylic acid. Position-specific exchanges are crucial to represent this superfamily accurately.
```Python
Res1 = StructureMotifResidue("A", "1", 162, ["LYS", "HIS"])
Res2 = StructureMotifResidue("A", "1", 193)
Res3 = StructureMotifResidue("A", "1", 219)
Res4 = StructureMotifResidue("A", "1", 245, ["GLU", "ASP", "ASN"])
Res5 = StructureMotifResidue("A", "1", 295, ["HIS", "LYS"])

ResList = [Res1, Res2, Res3, Res4, Res5]

query = StructMotifQuery(entry_id="2MNR", residue_ids=ResList)

list(query())
```

## Sessions

The result of executing a query (either by calling it or using `exec()`) is a
`Session` object. It implements `__iter__`, so it is usually treated just as an
iterator of IDs.

Paging is handled transparently by the session, with additional API requests made
lazily as needed. The page size can be controlled with the `rows` parameter.
```python
first = next(iter(query(rows=1)))
```
### Progress Bar

The `Session.iquery()` method provides a progress bar indicating the number of API
requests being made. It requires the `tqdm` package be installed to track the
progress of the query interactively.
```python
results = query().iquery()
```
