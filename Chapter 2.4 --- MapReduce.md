
## MapReduce

* MapReduce is a programming model for compute jobs.
* A **programming model** is a specification for (and/or implementation of) a paradigm that involves concepts distinct from those in the language calling it. 
* E.g. MapReduce is conceptually distinct from the C programming language, or Scala, and so on. This distinguishes it from a simple library!
* Compute jobs that are organized using the MapReduce framework are automatically parallelizable in way that is optimized for large-scale, distributed systems.
* And it is flexible. You can apply MapReduce to a vast range of compute jobs.
* It was introduced by a Jeffrey Dean paper years ago, and was the primary programming model for large-scale compute jobs at Google for a long time.
* In the open source community it is implemented via Apache Hadoop and the related ecosystem of tools.
* More recent work is pushing past the boundaries imposed by MapReduce, however!


* Large data jobs tends to involve simple computations that get performed against extremely large amounts of data.
* When the data volume is large enough that a single machine cannot easily handle it, many machines are used for processsing.
* With an a la carte system, the end result is a very small amount of simple computational code, and a very large amount of boilerplate dealing with parallelization, fault-tolerance, data distribution, and load balancing.
* The MapReduce system hides these messy details from you behind a simple abstraction.


* Each MapReduce job consists of a map and a reduce.
* In the **map** step, individual records (modeled as key-value pairs) are processed and turned into arbitrarily many intermediate key-value pairs.
* In the **reduce** step, pairs with the same key are co-processed into some final result.
* In both the map and reduce steps, the operations are atomic to individual data structures and are presented via an iterator. This is what makes this tractable memory-wise!
* For example, you could implement a simple word counter by emitting key-value pairs of the form $\\{word \\: : \\: 1\\}$, then writing a reduce function that sums across these key-value pairs (resulting a word-by-word sum of 1s, or a count!).
* In reality, the reduce step requires data with the same key to be colocated. Thus there is actually a third step: shuffle, which moves the intermediate mapped representations with the same key to the same memory space or node.
* Thus it might be more accurate to say MapShuffleReduce.


* The operational implementation described in the paper is as follows:
1. The size of the data is determined and divided by user-specified memory partition sizes. A mapper split size $M$ is calculated, along with a reducer split size $R$.
2. That many separate MapReduce processes are spun up on the cluster or machine. One process is the master, and it assigns work to the rest of the processes.
3. Map workers that are told by the master what data to work on parse the requisite key-value pairs out of the data, and start running the map operation on them.
4. The map workers hold the outputs in memory, but periodically flush them to disc in $R$ partitions (partitioning is handled by the partitioner). The location of these partitions is sent to the master.
5. The master assigns the partitions as they come in to reducers. The reducer reads that data in and sorts it.
6. The reducer applies the reduction operation to the data. The result of the reduction is appended to an output file when the reducer is done operating.
8. Once all of the reducers have finished running, the MapReduce system wakes up the overlying user program.


* Fault tolerance is provided via health polling from the master. If the master does not recieve a response from a polled worker in enough time, it marks it failed and moves that job onto another worker.
* Importantly, work that was already performed (or will be performed) by the failing worker is replicated by the reassigned worker. This is because the written-to resource is a local disc inaccessible to the master.
* The additional complexity of making the disc accessible might not be worth it!
* Reducers that were in the process of reading from the old map worker stop, and begin listening for input from the new map worker instead.
* Master failures are considered unlikely, and there is no provisioning for them in the original MapReduce paper. However, a periodic checkpointing scheme is described as being easy-to-implement.
* Hadoop implements safeties for this case.
* Note that if a worker node failure causes a reassignment, and the worker node eventually completes its job and signals the master about this, that signal is ignored.


* Locality is achieved by the placement of map and reduce processes near to data being mapped or reduced.
* This seems to be a system-specific implementation detail.


* Unplanned machine resource limitations are dealt with via back-up mechanism.
* The problem is that due to a host of reasons, but most often resource contention, machines may run code slower than expected.
* These stragglers will slow down overall execution time.
* The Google MapReduce system solves this by brute-forcing it, pretty much. When the job is close to completion, the remaining tasks are spun up on new machines.
* Now the first machine that completes the instruction (whether the original or the new backup) sets the standard. The job from the second machine is ignored.
* You could obviously increase the concurrency and/or back up at an earlier save point to further increase the resiliance and time-performance of the system, albeit at great computational cost.


* Some optimizations follow.
* You could specify a better partitioning function. The default is a hash-based one, but if you would like to have data grouped by, say, URL host, then a custom partitioning function may be specified and used.
* You may optionally gaurantee that the map and reduce jobs process and emit data in sort order.
* Transfering the data from the mapper process to the reducer process involves a network trip. To reduce the volume of network travel, you may optionally specify a **combiner**.
* A combiner partially reduces the response (for example, by throwing out non-interesting keys or creating an `OTHER` category or something, a need I am very familiar with!). It runs on the same machine as the map process, and is run on the map result *before* emission to the reducer.
* This can significantly reduce network resource utilization.
* The implementation describes assumes inputs are key-value pairs, but you can implement a seperable **reader** for any other input data type.
* A sequentially executing MapReduce implementation is a good thing to have, as it allows you to more easily debug otherwise asynchronous jobs.


* Hadoop is the main open source implementation of MapReduce, and what's used in production really.
* However, MapReduce is pretty old now. It's getting replaced by Spark

