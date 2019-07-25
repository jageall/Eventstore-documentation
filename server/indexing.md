---
outputFileName: index.html
---

# Indexing

Event Store stores indexes separately from the main data files accessing records by stream name.

## Overview

Event Store creates index entries as it processes commit events. It holds these in memory (called _memtables_) until the `MaxMemTableSize` is reached and then persisted on disk in the _index_ folder along with an index map file.
The index files are uniquely named, and the index map file is called _indexmap_. The index map describes the order and the level of the index file as well as containing the data checkpoint for the last written file, the version of the index map file and a checksum for the index map file. The index files are referred to a _PTables_ in the logs.

Indexes are sorted lists based on the hashes of stream names. To speed up seeking the correct location in the file of an entry for a stream, midpoints are kept to relate the stream hash to the physical offset in the file.

As Event Store saves more files, they are automatically merged together whenever there are more than 2 files at the same level into a single file at the next level. Each index entry is 24 bytes and the index file size is approximately 24Mb per 1M events.

Level 0 is the the level of the _memtable_ that is kept in memory. Generally there will only be 1 level 0 table unlees multiple level 0 tables are produced during an ongoing merge operation.

Assuming the default `MaxMemTableSize` of 1M, the index files by level are:

| Level | Number of entries | Size          |
| ----- | ----------------- | ------------- |
| 1     | 1M                | 24MB          |
| 2     | 2M                | 48MB          |
| 3     | 4M                | 96MB          |
| 4     | 8M                | 192MB         |
| 5     | 16M               | 384MB         |
| 6     | 32M               | 768MB         |
| 7     | 64M               | 1536MB        |
| 8     | 128M              | 3072MB        |
| n     | (2^n) * 1M        | (2^n-1) * 24Mb|

Each index entry is 24 bytes and the index file size is approximately 24Mb per M events. 

### Configuration Options

The configuration options that effect indexing are:

- `Index` : where the indexes are stored
- `MaxMemTableSize` : how many entries to have in memory before writing out to disk
- `IndexCacheDepth` : sets the minimum number of midpoints to calculate for an index file
- `SkipIndexVerify` : skips reading and verification of PTables during start-up
- `MaxAutoMergeIndexLevel` : the maximum level of index file to merge automatically before manual merge
- `OptimizeIndexMerge` : Bypasses the checking of file hashes of indexes during startup and after index merges (allows for faster startup and less disk pressure after merges)

See [Command line arguments](#command-line-arguments) for how to specify these options.

#### Index

`Index` effects the location of the index files. We recommend you place index files on a separate drive to avoid competition for IO between the data, index and log files.

#### MaxMemTableSize

`MaxMemTableSize` effects disk IO when Event Store writes files to disk, index seek time and database startup time. The default size is a good tradeoff between low disk IO and startup time. Increasing the `MaxMemTableSize` results in longer database startup time because before a node can come online it has to read through the data files from the last position in the `indexmap` file and rebuild the in memory index table.

Increasing `MaxMemTableSize` also decreases the number of times Event Store writes index files to disk and how often they are merged together. When they are written to disk or merged, IO increases. It also reduces the number of seek operations when stream entries span multiple files as each file needs to be searched for the stream entries. This effects streams that are written to over longer periods of time more than streams where events are written in a short time, where time is measured as a function of the number of events created.  This is because streams written to over longer time periods are more likely to have entries in multiple index files.

#### IndexCacheDepth

`IndexCacheDepth` effects the how many midpoints Event Store calculates for an index file which effects file size slightly, but can effect lookup times significantly. Looking up a stream entry in a file requires a binary search on the midpoints to find the nearest midpoint, and then a seek through the entries to find the entry or entries that match. Increasing this value decreases the second part of the operation and increase the first for extremely large indexes.

The default value of 16 results in files up to about 1.5GB in size being fully searchable through midpoints. After that a maximum distance between midpoints of 4096 bytes for the seek, which is buffered from disk, up until a maximum level of 2TB where the seek distance starts to grow. Reducing this value can relieve a small amount of memory pressure in highly constrained environments. Increasing it causes index files larger than 1.5GB, and less than 2TB to have more dense midpoint populations which means the binary search will be used for longer before switching back to scanning the entries between. The maximum number of entries that would be scanned in this way is distance/24b so with the default setting and a 2TB index file this would be approximately 170 entries. Most clusters should not need to change this setting.

#### SkipIndexVerify

`SkipIndexVerify` skips reading and verification of index file hashes during startup. Instead of recalculating midpoints when Event Store reads the file, the midpoints are read directly from the footer of the index file. You can set `SkipIndexVerify` to `true` to reduce startup time in exchange for the acceptance of a small risk that the index file is corrupted. This corruption could lead to a failure if you read the corrupted entries, and a message saying the index needs to be rebuilt. This can be safely switched on for ZFS on Linux as the filesystem takes care of file checksums.

In the event of corruption indexes will be rebuilt by reading through all the chunk files and recreating the indexes from scratch.

#### MaxAutoMergeIndexLevel

`MaxAutoMergeIndexLevel` allows you to specify the maximum index file level to automatically merge. By default all levels are merged. Depending on the specification of the host running Event Store, at some point index merges will use a very large amount of disk IO.

For example, merging 2 level 7 files results in at least 3072MB of reads (2 \* 1536MB) and 3072MB writes while merging 2 level 8 files together results in at least . Setting MaxAutoMergeLevel to 7 will allow all levels up to and including level 7 to be automatically merged, but to merge the level 8 files together, you need to trigger a manual merge needs. This manual merge allows better control over when these larger merges happen and which nodes they happen on. Due to the replication process, all nodes tend to merge at about the same time.

#### OptimizeIndexMerge

`OptimizeIndexMerge` allows faster merging of indexes when Event Store has scavenged a chunk. This option has no effect on unscavenged chunks. When Event Store has scaveneged a chunk, and this option is set to `true`, a bloom filter is used before reading the chunk to see if the value exists before reading the chunk to make sure that it still exists.

## Indexing in depth

For general operation of EventStore the following information is not critical but useful for developers wanting to make changes in the indexing subsystem and for understanding crash recovery and tuning scenarios.

### Index map files

_Indexmap_ files are text files made up of line delimited values. The line delimiter varies based on operating system, so while _indexmap_ files may be considered valid when being transferred between operating systems if changes are being made to fix an issue (for example, disk corruption) it is best to make sure that the changes are being made on the same operating system as the cluster.

The _indexmap_ structure is as follows:
 - hash - an md5 hash of the rest of the file
 - version - the version of the _indexmap_ file
 - checkpoint - the maximum prepare/commit position of the persisted _ptables_
 - maxAutoMergeLevel - either the value of [MaxAutoMergeLevel](MaxAutoMergeLevel) or `int32.MaxValue` if [MaxAutoMergeLevel](MaxAutoMergeLevel) was not set. This is primarily used to detect increases in [MaxAutoMergeLevel](MaxAutoMergeLevel) which is not supported.
 - _ptable_,level,index * - List of all the _ptables_ used by this index map with the level of the _ptable_ and it's order

_Indexmap_ files are written to a temporary file and then the original is deleted and the temporary file renamed.  Event Store will attempt this 5 times before failing. Because of the order this operation can only really fail if there is an issue with the underlying file system or the disk is full. This is a 2 phase process and in the unlikely event of a crash during this process, EventStore will recover by rebuilding the indexes using the same process that is used if it detects corrupted files during startup.

### Writing and Merging of index files

Merging _ptables_, updating the _indexmap_ and persisting _memtable_ operations are done on a background thread. These operations are performed on a single thread with any additional operations being queued and completed later. These operations are run on a thread pool thread rather than a dedicated thread. Generally there will only be one operation queued at a time, but if merging to _ptables_ at one level causes 2 tables to be available at the next level, then the next merge operation will be immediately queued. While merge operations are occuring, if there are large enough numbers of events being written, 1 or more _memtables_ may be queued for persistence. The number of pending operations is logged.

For safety _ptables_ being merged are only deleted after the new _ptable_ has been persisted and the  _indexmap_ has been updated. In the event of a crash EventStore recovers by deleting any files not in the _indexmap_ and reindexing from the prepare/commit position stored in the _indexmap_ file.

#### Manual Merging
If the [maximum level](MaxAutoMergeIndexLevel) for automatically merging indexes has been set, then merging indexes above this level need to be triggered manually. This can be done by using curl or a similar tool to post to the /admin/manualmerge endpoint using appropriate credentials or using the ES-CLI tool that is available with commercial support. 

Triggering a manual merge will cause all tables that have a level equal to the maximum merge level or above to be merged into a single table.  If there is only 1 table at the maximum level or above, no merge will be performed.

### Tuning

For most Event Store clusters, the default settings are sufficient to provide consistent and good performance. For clusters with larger numbers of event or those that run in constrained environments the configuration options allow for some tuning to meet operational constraints.

The most common optimization needed is to set a `MaxAutoMergeLevel` to avoid large merges occuring across all nodes at approximately the same time.  Large index merges use a lot of IOPS and in IOPS constrained environments it is often desirable to have better control over when these happen. Because increasing this value requires an index rebuild you should start with a higher value and decrease until the desired balance between triggering manual merges (operational cost) and automatic merges (IOPS) cost.  The exact value to set this to will vary between environments due to IOPS generated by other operations such as read and write load on the cluster.

For example, a cluster with 3000 256b IOPS can read/write about 0.73Gb/sec (This level of IOPS represents a fairly small cloud instance). Assuming sustained read/write throughput of 0.73Gb/s. When an index merge of level 7 or above starts, it will consume as many IOPS up to all on the node until it completes. Because Event Store has a shared nothing architecture for clustering this operation is likely to cause all nodes to appear to stall simultaneously as they all try and perform an index merge at the same time. By setting `MaxAutoMergeLevel` to 6 or below this can be avoided and the merge can be run on each node individually keeping read/write latency in the cluster consistent.


<!-- # todo: the 64 bit index bits should probably come under this indexing doc -->
