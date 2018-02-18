---
title: "Redis Learning Notes"
categories: Redis 
---
## Cluster
### Redis cluster 101
* automatically split your dataset among multiple nodes.
* continue operations when a subset of the nodes are experiencing failures

### Data sharding

* 16384 total hash slots
* hash slot = CRC16(key) % 16384
* Every node is responsible subset of hash slots. Need manually assign.
* Use **hash tag** to keys to be the same hash slot. For example, **this{foo}key** and **another{foo}key**

### Master slave model

Slave will be promoted to the master if one master is down to keep cluster running.

### Consistency guarantee

* **No strong consistency**. Cluster may lose writes acknowledged.
* Due to asynchronous replication, the slave which behind master might be promoted to be new master

### How to config

```
port 7000
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
```

### Design goals

1. High performance and linear scalability up to 1000 nodes
2. Acceptable degree of write safety: the system tries (in a best-effort way) to retain all the writes originating from clients connected with the majority of the master nodes.
3. Availability: Redis Cluster is able to survive partitions where the majority of the master nodes are reachable and there is at least one reachable slave for every master node that is no longer reachable.

### Eviction

### Eviction policies 

* **noeviction**: return errors when the memory limit was reached and the client is trying to execute commands that could result in more memory to be used (most write commands, but DEL and a few more exceptions).
* **allkeys-lru**: evict keys by trying to remove the less recently used (LRU) keys first, in order to make space for the new data added.
* **volatile-lru**: evict keys by trying to remove the less recently used (LRU) keys first, but only among keys that have an expire set, in order to make space for the new data added.
* **allkeys-random**: evict keys randomly in order to make space for the new data added.
* **volatile-random**: evict keys randomly in order to make space for the new data added, but only evict keys with an expire set.
* **volatile-ttl**: evict keys with an expire set, and try to evict keys with a shorter time to live (TTL) first, in order to make space for the new data added.

### Approximated LRU algorithm

```
maxmemory-samples 5
```


## Metrics

### Memory

* **used_memory**: total number of bytes allocated by Redis using its allocator (either standard libc, jemalloc, or an alternative allocator such as tcmalloc
* **used_memory_human**: Human readable representation of previous value
* **used_memory_rss**: Number of bytes that Redis allocated as seen by the operating system (a.k.a resident set size). This is the number reported by tools such as top(1) and ps(1). 
  - rss >> used means fragmentation.
  - used >> rss means part of memory swapped, expect significant latency
  - Redis will not always free up (return) memory to the OS when keys are removed. So **maxmemory** should be based on peak memory usage. However allocators are abke to reuse free memory. Because of all this, fragmentation ratio is not reliable when peak memory >>> currently used memory.
* **used_memory_peak**: Peak memory consumed by Redis (in bytes)
* **used_memory_peak_human**: Human readable representation of previous value
* **used_memory_lua**: Number of bytes used by the Lua engine
* **mem_fragmentation_ratio**: Ratio between used_memory_rss and used_memory
* **mem_allocator**: Memory allocator, chosen at compile time

## Persistency

### RDB: point-in-time snapshots of dataset at specified interval

#### Pros
compact, good for disaster recovery, little performance overhead, faster restarts

#### Cons
certain data loss, fork() often. (AOF can tune rewrite fork frequency)

### AOF: logs every write operation, that will be played again at server startup

#### Pros
* more durable, control fsync policy
* append-only file, no seek, no corruption, could be fixed
* rewrite big AOF in background

#### Cons
bigger than RDB, slower, less robust

#### What should I use

* Use both if you want data safety like PostgreSQL
* Use RBD alone if you live with minutes of data loss
* Discourage to use AOF alone

#### How config RDB

Dump every 60 seconds if at least 1000 keys changed
```
save 60 1000 
```

#### How RDB works
1. Redis forks. We now have a child and a parent process.
2. The child starts to write the dataset to a temporary RDB file.
3. When the child is done writing the new RDB file, it replaces the old one.

#### How enable AOF

```
appendonly yes
```
  
#### How AOF log rewritting works

1. Redis forks, so now we have a child and a parent process.
2. The child starts writing the new AOF in a temporary file.
3. The parent accumulates all the new changes in an in-memory buffer (but at the same time it writes the new changes in the old append-only file, so if the rewriting fails, we are safe).
4. When the child is done rewriting the file, the parent gets a signal, and appends the in-memory buffer at the end of the file generated by the child.
5. Profit! Now Redis atomically renames the old file into the new one, and starts appending new data into the new file.

#### How durable is the append only file?
1. fsync every time new command appended. Very very slow, very saft
2. fsync every second. Fast enough.
3. Never fsync, rely on OS. faster and less safer.

## Replication

### Three main mechanism:

1. When master and slave are well-connected, master send a stream of commands.
2. When link breaks, slave reconnects and proceeds with partial resynchronization: obtain part of stream of commands
3. When partial resynchronization is impossible, slave ask for full resynchronization. Master send snapshot to slave, then send stream of commands.

### Important facts

* Asynchronous replication
* Replication is non-blocking on the master side.
* A master can have multiple slaves.
* Slave are able to accept connections from other slaves.

### Allow writes only with N attached replicas

* slaves ping the master every second, acknowledging the amount of replication stream processed.
* masters will remember the last time it received a ping from every slave.
* The user can configure a minimum number of slaves that have a lag not greater than a maximum number of seconds.

```
min-slaves-to-write <number of slaves>
min-slaves-max-lag <number of seconds>
```

### How deal with expiration

1. Slave don't expire keys, master send **DEL** after key expire



