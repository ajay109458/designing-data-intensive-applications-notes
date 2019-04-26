## Data Model

* A data model defines how data is organized and persisted in a system.
* They are fundamentally a layer of abstraction. The choice of data model provides a unified API, one that is good for certain things and bad for others and which hides the implementation and performance details lower down.

## Types of Data Model
* The **relational model** is the most standard data model. Classical relational databases follow this model. In the relational model data is organized in terms of relations (tables) and tuples (rows).
* A **key-value store** is an alternative construction, where each element has a unique key corresponding with some chunk of data as the store.
* Technically speaking the \"value\" in key-value can be any abstract type, from a string all the way to binary blob. The **document model** is a subtype of key-value stores in which each of the data chunks is forced to be a document.
* An example of a key-value store is Redis. An example of a document-oriented database (a database using the document model) is MongoDB.
* The **graph model** is a data model that treats data as blobs of information connected by edges representing relationships.
* Some relational databases may implement document model support as well, on the side.
* Graph databases can be implemented using relational databases, or key-value stores, or some other underlying system.

## Compare Relational V/S NoSQL databases
* The biggest issue with the relational model that so-called \"NoSQL\" models try to address is **object-impedence mismatch**. The structure of the data in the database doesn't look that much like the structure of the data as it is applied in the program, forcing the programmer to have to deal with two different memory structures.
* For a broad range of applications, document databases are good at locality. Data that is consumed together is located close together within the document schema. In a relational model by contrast useful data can often be several joins away, unless you start denormalizing.
* The strength of relational databases is in joins. If your application needs to perform joins often, then relational models are still the gold standard. Document stores do not intrinsically support these joins so you will have to reimplement them at the application level. If you don't need to perform joins very often, non-relational models are more viable.
* Document databases are more flexible than relational databases because they are **schema-on-read**, whereas relational databases are **schema-on-write**. This just-in-time schema dependency can be a boon or a burden, depending on your product and development cycle.
* When a modification to the data is made, document databases typically have to reload the entire document. If there are many write operations, document database performance suffers. When using a document store you want to have small documents that are consumed nearly completely and modified in large segments.

## Graph dataabase 
* Graph databases are implemented using either a property graph or a triples approach. The former is a direct translation of graph theory, while the latter is extracted from work on the semantic web.\n",
* The property graph is based on nodes (storing data in key-value pairs) with sets of edges leading towards and away from them.
* Edges are similarly key-value pair stores, which additionally have a label property as well as (optionally) information on direction: what node is the head and what node the tail (e.g. directed graph or undirected graph?).
* You can naively implement a graph database yourself by creating two relational tables, one with edges and one with nodes. Connections are just foreign keys!
