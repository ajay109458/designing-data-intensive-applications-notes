
# TrailDB

## Mission statement
* TrailDB is a database for (1) exploring time-series data (2) with locality (3) optimized for analytics.
* First, time-series data. The TrailDB data model has good performance on time-based tasks, like computing window functions or period aggregations.
* Second, locality. A time-series database is optimized for working with time-series data. However, a regular time-series database does not impose any logical structure on or otherwise keep events from the same emitter close together. TrailDB does, which makes it faster and more workmanlike for such queries.
* Finally, it is optimized for analytics. The database components are supposed to be simple and portable, at the cost of feature surface.

## Data model
* Database is a single file, as in SQLite.
* Once created, the file is read-only. Thus you can use this architecture for analytics but not as a production service!
* Thus TrailDB strongly asserts what is known as referential transparency.
* Data is organized along two axes, users (I will call them emiters here, as this idea is more general than users) and trails.
* The architecture is a key-value store with emitter IDs (user IDs) as the keys (keys are 128-bit values) and the trails as the values.
* Trail consist of sequences of timestamped events. They are organizing in ascending time order.
* One event contains multiple items. Events consist of a field ID and a value ID. The first field ID is always `NULL`. Items are encoded on disc as a 64-bit integer.
* Thus everything is represented on disk as an integer. Notice that this means that the data is highly efficiently stored (everything is 64-bit integers on disc, plus some metadata), but limits the maximum cardinality of your data.
* Going further, everything is represented in a compressed way. The manner in which the data is compressed is moderately complex. It takes advantage of regularities in the data, and is done in blocks of stuff and not on everything all together (in contrast to GZip), so that you only decompress what you need.

## Operations
* Looking up a trail by ID is $O(log N)$. This is a win over time-series databases, which can only return individual timestamped events at this speed.
* Iteration over the keys is also fast.
* Data is stored in integer format, and parsing into e.g. string fields can be done via included utilities. Most databases have a similar store-by-ID implementation, but do not expose this fact anywhere. TrailDB chooses to expose you to this because they believe you can leverage som speed advantage from it, if you pick IDs intelligently.
* You can build the database and you can merge two databases from unordered streams of data. Merging databases is an efficient operation.
* Maping a string to an item is $O(L)$, where $L$ is the number of possible values that item can take on.
* Cursors are cheap.
* TrailDB implements virtual views as \"event filters\". You can specify at query time that only records matching certain conditions be returned. Event filters expedite sub-selection operations.
* You can also get materialized views, e.g. export event filters to separate TrailDB files.
* You can define a multi-cursor that connects to multiple TrailDB instances. This is useful because a common "
