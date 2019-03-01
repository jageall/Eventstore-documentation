---
outputFileName: index.html
---

# Indexing

Event Store stores indexes separately from the main data files accessing records by stream name.

## Overview

Event Store creates index entries as clients commit events. It holds these in memory until the `MaxMemTableSize` is reached and then persisted on disk in the _index_ folder along with an index map file.
The index files are uniquely named, and the index map file is called _indexmap_. The index map describes the order and the level of the index file as well as containing the data checkpoint for the last written file, the version of the index map file and a checksum for the index map file. The index files are referred to a _PTables_ in the logs.

Indexes are sorted lists based on the hashes of stream names. To speed up seeking the correct location in the file of an entry for a stream, midpoints are kept to relate the stream hash to the physical offset in the file.

As Event Store saves more files, they are automatically merged together whenever there are more 2 files at the same level into a single file at the next level. Each index entry is 24 bytes and the index file size is approximately 24Mb per M events

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
| n     | (2^n) \_ 1M       | (2^n-1)\_24Mb |

Each index entry is 24 bytes and the index file size is approximately 24Mb per M events

## Configuration Options

The configuration options that effect indexing are:

- `Index` : where the indexes are stored
- `MaxMemTableSize` : how many entries to have in memory before writing out to disk
- `IndexCacheDepth` : sets the minimum number of midpoints to calculate for an index file
- `SkipIndexVerify` : skips reading and verification of PTables during start-up
- `MaxAutoMergeIndexLevel` : the maximum level of index file to merge automatically before manual merge
- `OptimizeIndexMerge` : Bypasses the checking of file hashes of indexes during startup and after index merges (allows for faster startup and less disk pressure after merges)

See [Command line arguments](#command-line-arguments) for how to specify these options.

## Merging

## Tuning

### Index

`Index` effects the location of the index files. We recommend you place index files on a separate drive to avoid competition for IO between the data, index and log files.

### MaxMemTableSize

`MaxMemTableSize` effects disk IO when Event Store writes files to disk, index seek time and database startup time. The default size is a good tradeoff between low disk IO and startup time. Increasing the `MaxMemTableSize` results in longer database startup time because before a node can come online it has to read through the data files from the last position in the `indexmap` file and rebuild the in memory index table.

`MaxMemTableSize` also decreases the number of times Event Store writes index files to disk and how often they are merged together. When they are written to disk or merged, IO increases. It also reduces the number of seek operations when stream entries span multiple files as each file needs to be searched for the stream entries. This effects streams that are written to over longer periods of time more than streams where events are written in a short time, where time is measured as a function of the number of events created.

### IndexCacheDepth

`IndexCacheDepth` effects the how many midpoints Event Store calculates for an index file which effects file size slightly, but can effect lookup times significantly. Looking up a stream entry in a file requires a binary search on the midpoints to find the nearest midpoint, and then a seek through the entries to find the entry or entries that match. Increasing this value decreases the second part of the operation and increase the first for extremely large indexes.

The default value of 16 results in files up to about 1.5GB in size being fully searchable through midpoints. After that a maximum distance between midpoints of 4096 bytes for the seek, which is buffered from disk, up until a maximum level of 2TB where the seek distance starts to grow. Reducing this value can relieve a small amount of memory pressure in highly constrained environments. Increasing it causes index files larger than 1.5GB, and less than 2TB to have more dense midpoint populations.

### SkipIndexVerify

`SkipIndexVerify` skips reading and verification of index file hashes during startup. Instead of recalculating midpoints when Event Store reads the file, the midpoints are read directly from the footer of the index file. You can set `SkipIndexVerify` to `true` to reduce startup time in exchange for the acceptance of a small risk that the index file is corrupted. This corruption could lead to a failure if you read the corrupted entries, and a message saying the index needs to be rebuilt. This can be safely switched on for ZFS on Linux as the filesystem takes care of file checksums.

### MaxAutoMergeIndexLevel

`MaxAutoMergeIndexLevel` allows you to specify the maximum index file level to automatically merge. By default all levels are merged. Depending on the specification of the host running Event Store, at some point index merges will use a very large amount of disk IO.

For example, mergin 2 level 7 files results in at least 3072MB of reads (2 \* 1536MB) and 3072MB writes. Setting this to level 7 will be automatically merged, but to merge the level 7 files together, you need to trigger a manual merge needs. This manual merge allows better control over when these larger merges happen and which nodes they happen on. Due to the replication process, all nodes tend to merge at about the same time.

### OptimizeIndexMerge

`OptimizeIndexMerge` allows faster merging of indexes when Event Store has scavenged a chunk. This option has no effect on unscavenged chunks. When Event Store has scaveneged a chunk, and this option is set to `true`, a bloom filter is used before reading the chunk to see if the value exists before reading the chunk to make sure that it still exists.

<!-- # todo: the 64 bit index bits should probably come under this indexing doc -->
