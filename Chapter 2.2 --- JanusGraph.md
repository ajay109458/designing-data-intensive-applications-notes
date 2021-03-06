
# JanusGraph

## Mission statement

* JanusGraph is an open-source, Linux Foundation supported graph database.
* It's meant to be a completely open source alternative to the more popular Neo4J, which is corporate copyleft.
* It's designed to scale horizontally across crazy numbers of compute nodes.
* It supports Hadoop job organization and graph queries using the Gremlin declarative structured query language (which has an Apache-developed Python binding, in addition to a as-you-would-expect Java binding).
* It does not come with a backing store baked-in; you need to select one yourself.
* The backing store in production may be either Apache Cassandra, which is continuously available but inconsistent, or HBase, which is strongly consistent but doesn't have high availability guarantees (cf. CAP theorem).
* An Oracle Berkeley DB Java Edition (BerkeleyDB JE). This should only be used for testing and exploration purposes however.
* Indices may also be used. These will speed up and/or enable more complex queries. The options are Elasticsearch, Solr, and Lucerne.
* You can use it either monolithically (queries are executed within the same JVM) or as a service.
* JanusGraph seems to have a so-so but not too-large userbase.
* It seems to be backed by IBM, Google, and the Linux Foundation.
* AWS is the only hosted graph database provider as far as I can tell. They provide AWS Neptune, which is...a closed source unknown. OK.

## Configuration
* A JanusGraph graph database cluster consists of one or more JanusGraph **instances**.
* Instances are built using a **configuration**.


* A console is included in JanusGraph. This can be used to e.g. open graphs and inspect them using the built-in Gremlin tooling.
* The configuration of the graph and the location of the graph files are specified by a file, which you can at run-time by whatever you use to initialize JanusGraph. From the console this looks like so:

        graph = JanusGraphFactory.open('path/to/configuration.properties')

* Dealing with configuration details gets complicated quickly. A temporary-most configuration is:

        storage.backend=berkeleyje
        storage.directory=/tmp/graph


* At a minimum, a storage backend must be specified. Optionally, a indexing backend and some cache configuration may be provided, if performance is a concern.


* On its own JanusGraph is just a bunch of JAR files. It may be used on the same thread as a program running on the JVM, or it may be ran as a persistent server process.
* Doing the latter means deploying a process known as the JanusGraph Server. JanusGraph Server uses the Gremlin Server as its service layer.
* Unlike the storage configuration, this configuration is specified (1) prior to execution and (2) at a specific file location, `./conf/gremlin-server` (with respect to the JanusGraph folder). A simple server configuration is:

        graphs: {
          graph: conf/janusgraph-berkeleyje.properties
        }
        plugins:
          - janusgraph.imports


* Graphs bound to the server point to specific configuration files, and the server proceeds to use that configuration file to set itself up.
* To actually int, run `bin/janusgraph.sh start`.


* The configuration layer allows applying configurations in five different scopes (roughly speaking, from local to a single instance to global for the entire cluster).
* The configuration of the first instance is inherited from the configuration file.
* The configuration of subsequent instances may be modified by calling on certain functions in the configuration layer of JanusGraph.
* Old instances do not inherit changes to the configuration details (e.g. there is no built-in rolling update support).
* To update configuration details for an entire cluster, you need to manually pause the process: close all but one instance, ensure that said instance is offline and done processing transactions, apply the update, and rescale back up.
* This is of course very clunky. In practice you're probably better off scaling with changes at the cluster management layer.


* The recommend server gateway is a WebSocket connection. A custom subprotocol is used for communications with JanusGraph, which again comes from Gremlin.
* The quickest way to test for a connection is with Gremlin Console. You can `:remote connect tinkerpop.server conf/remote.yaml`. In this command, `tinkerpop.server` is the Gremlin plug-in that enables server connecting, and `conf/remote.yaml` is a pointer to the file (it doesn't have to be this specific name) that provides configuration details for Gremlin's connection.
* It is also possible to configuration communication via an HTTP API. However, this requires a lot of additional work, as far as I can tell, so I'm not sure why you would want to do that.

## Data model

* JanusGraph uses a graph theory -style vertex-edge schema (as opposed to a semantic web -style triple schema).
* It can be either schema-on-read or schema-on-write. Schema-on-write, e.g. specifying and following an explicit schema, is heavily encouraged.
* You can (should?) turn off implicit schemaby specifying `schema.default=none` in the cluster configuration file.
* All schema details are handled via the management layer (the same one that controls instance configuration). So, if you go the explicit schema route, you want to define an explicit schema before creating any instances.


* An important schema configuration detail is **edge label multiplicity**. Edges between vertices may be one-to-one (simple), one-to-many, many-to-one, or many-to-many.
* Graph databases always handle directed graphs.
* One-to-one and many-to-many relationships are easy to understand.
* A many-to-one relation specifies that there may be at most one outgoing edge but unlimited incoming edges. A `IS CHILD OF` relation is a good example (a single person has one parent, but one parent can have many children).
* A one-to-many relation is the opposite. A `PARENT OF` relation, which is the inverse of the relation above, is a good example of such.


* Both vertices and edges have a label and a key-value store of properties.
* Edge labels and property keys are jointly known as relation types, and must be uniqueley named. So you cannot have both an edge and a property with the same name.
* Properties on verticles and edges are key-value pairs having a certain type.
* The list of types is about what you would expect from a database. There's no blob, but there is a geoshape.
* The values your keys map to may be singletons, lists, or sets. This is **property key cardinality**, and can be configured at schema definition time as well, obviously.
* Edge labels are the only requirement, all of vertex labels, vertex properties, and edge properties are optional.
* The list of valid vertex labels is another schema configuration detail that can be specified at start time.


* Schema changes that do not touch existing data propogate safely. Not sure if there's a way to tell if it's hit all of your instances yet.
* Schema changes that touch existing data (e.g. database migrations) will fail. You need to perform a database migration, which in graph theory land means a batch graph transformation, for those to go through. That can only be done offline.
* The one exception appears to be edge label changes. Edge label updates can happen online, but will not touch existing vertices or edges.
* You an also add indexes (next section) on the fly.


* Ghost vertices are a problem on eventually consistent backends. Sounds a lot like something I read could happen in MongoDB:

    > When the same vertex is concurrently removed in one transaction and modified in another, both transactions will successfully commit on eventually consistent storage backends and the vertex will still exist with only the modified properties or edges. This is referred to as a ghost vertex. It is possible to guard against ghost vertices on eventually consistent backends using key uniqueness but this is prohibitively expensive in most cases. A more scalable approach is to allow ghost vertices temporarily and clearing them out in regular time intervals.


## Indexing

* Graph databases, like regular transactional databases, can get significant speed boosts from using indices.


* Most graph queries start by looking at a list of vertices or edges, and you can speed these queries up by keeping a sorted list of such indices in memory.
* JanusGraph supports two index types: **composite** and **mixed**.
* Composite indices are multi-column B-tree indices. They are good for searching arbitrarily deep into an ordered list of attributes. For example, if you have an index built on $(A, B, C)$, where $A$ is the first indexed property and $C$ is the last. You can index $(A)$, $(A, B)$, $(A, B, C)$, but it'll be no use for $(B)$ or $(B, C)$ queries.
* Mixed indexes are just multiple single-column indices, best I can tell.
* Composite indices are built-in. Mixed indices require shelling out to an indexing backend like ElasticSearch.
* You can explicitly disable full graph scans with the `force-index` configuration option. This is recommended for large graphs at production scale.
* The details of how these indexes work, and thus what they can and can't do, are at the mercy of the index store you are using. That's a subject for another set of notes...



* Graph databases (and JanusGraph) can also have **vertex indices**. These are indices on the vertices of the graph. They are useful when performing vertex-centric queries, e.g. queries that have reason to scan all or a significant subset of the edges of specific vertices. Vertex indices speed up searching through these relationships.
* This is a useful property to have for graphs where vertices may have 100s or 1000s of edges.


* You can await the completion of an index creation in flight in a synchronous way using an `awaitGraphIndexStatus` hook in the management layer.

## Transactions

* JanusGraph is only ACID on the BerkeleyDB JE backend. Neither Cassandra nor HBase, the two backend options, provide this guarantee.
* **Transactional scope** is the behavior of pointers with respect to the database transaction.
* JanusGraph is obvious transactional. When defining a new vertex of edge, we may assign a reference to that vertex of edge to a variable in the language we are performing the operation in.
* Once the change is committed, if the variable pointer remains valid we state that it has \"transitioned transactional scope\". If it is no longer valid, that variable is lost because it has \"left transactional scope\".
* This is a non-trivial feature because the object reference has to reflect the value currently in the database. There may be a difference between commit time and update time (especially in an explicitly \"eventually consistent\" database), which can cause the object data and the database data to desync.
* JanusGraph will transition vertex references for you, but it will not transition edges.
* If you want to transition edges you have to do so manually.


* JanusGraph includes some logic for attempting to mitigate temporary transaction failures due to things like e.g. network hiccups. Eventually it will give up (configurable after how many retries).
* Permanent failures, like lock failures due to race conditions, may also occur. In these cases you will get a transaction error, which you will need to handle somehow.


* JanusGraph supports multi-threaded transactions. You can isolate a transaction to a specific thread. This can be used to speed up many (embarrassingly parallel) graph algorithms.
* You can also use thread isolation to avoid potential lock contention on long-running transactions where at least one of the operations has to lock the database.
* Wrap that sub-transaction into a separate thread, and commit that in the body of the transaction. This will force a (sort of a synchronous) lock and make the rest of the transaction respect it (still not 100% clear on this point; see 10.6 in https://docs.janusgraph.org/latest/tx.html).


## Caching

* JanusGraph offers multiple levels of caches. From closest to furthest they are transaction-level caches, database-level caches, and storage backend caches.
* The transaction level cache stores the (1) properties and (2) adjacency list of accessed vertices, and access locations in any indices, in memory.
* This cache is local to each transaction, e.g. it gets flushed at the end of the transaction.
* This cache is most useful for queries that hit the same or similar nodes many times, obviously. It's something to take advantage when designing transactions.
* The next layer of cache is the database-level cache. This cache retains data across multiple transactions, and only covers adjacency list access hotspots. The database-level cache can significantly speed up read-heavy workloads.
* Finally, the storage backend providers maintain (usually) their own data caching layer. This is typically more compressed, compacted, and much larger than the in-process one. It also manages memory outside of the JVM typically (it is \"off-heap\") so it is less prone to slowdown. But it is the slowest of the cache layers too, for the same reason.

## Logging

* JanusGraph includes a user transaction logging feature.
* This can be usedas a record of change (e.g. for log parsing), for downstream updates in a more complex system, and for defining triggers on certain events.
* These are useful because using the logs for this purpose, instead of querying the database directly, doesn't slow the process down."

