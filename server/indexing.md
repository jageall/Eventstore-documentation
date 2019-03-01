---
outputFileName: index.html
---
# Indexing

Event Store stores indexes seperately from the main data files accessing records by stream name.

## Overview

Index entries are created as events are committed to Event Store. These are held in memory until the _MaxMemTableSize_ is reached and then persisted on disk in the _index_ folder along with an index map file.
The index files are uniquely named with and the index map file is called `indexmap`. The index map describes the order and the level of the index file as well as containing the data checkpoint for the last written file, the version of the index map file and a checksum for the index map file. The index files are refered to a _PTables_ in the logs.

Indexes are sorted lists based on the hashes of stream names. To speed up seeking the correct location in the file of an entry for a stream, midpoints are kept to relate the stream hash to to the physical offset in the file.

As more files are saved, they are automatically merged together whenever there are more 2 files at the same level into a single file a the next level. Each index entry is 24 bytes and the index file size is approximately 24Mb per M events

Assuming the default _MaxMemTableSize_ of 1M, the index files by level will be:

| Level | Number of entries | Size          |
| ----- | ----------------- | ------------- |
| 1     |                1M |         24MB  |
| 2     |                2M |         48MB  |
| 3     |                4M |         96MB  |
| 4     |                8M |        192MB  |
| 5     |               16M |        384MB  |
| 6     |               32M |        768MB  |
| 7     |               64M |       1536MB  |
| 8     |              128M |       3072MB  |
| n     |        (2^n) * 1M | (2^n-1)*24Mb  |

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

`Index` effects the location of the index files. Placing index files on a seperate drive to avoid competition for IO between the data, index and log files and is recommended.

### MaxMemTableSize

`MaxMemTableSize` effects disk IO when the files are written to disk, index seek time and database startup time. The default size is a good tradeoff between low disk IO and startup time. Increasing the `MaxMemTableSize` will result in longer database startup time because before a node can come online it has to read through the data files from the last position in the `indexmap` file and rebuild the in memory index table. It will also decrease the number of times index files are written to disk and how often they are merged together. When they are written to disk or merged IO will be increased. It also reduces the number of seek operations when stream entries span multiple files as each file will need to be searched for the stream entries. This effects streams that are written to over longer periods of time more than streams where events are written in a very short time, where time is measured as a function of the number of events created.

### IndexCacheDepth

`IndexCacheDepth` this effects the how many midpoints will be calculated for an index file which effects file size slightly, but can effect lookup times significantly. Looking up a stream entry in a file requires a binary search on the midpoints to find the nearest midpoint, and then a seek through the entries to find the entry or entries that match.  Increasing this value will decrease the second part of the operation and increase the first for extremely large indexes. The default value of 16 results in files up to about 1.5GB being fully searchable through midpoints and after that a maximum distance between midpoints of 4096 bytes for the seek, which is buffered from disk, up until a maximum level of 2TB where the seek distance starts to grow. Reducing this value can relieve a small amount of memory pressure in highly constrained environments. Increasing it will cause index files larger than 1.5GB and less than 2TB to have more dense midpoint populations.

### SkipIndexVerify
`SkipIndexVerify` Skips reading and verification of index file hashes during startup. Instead of recalculating midpoints when the file is read, the midpoints are read directly from the footer of the index file. This can be switched on to reduce startup time in exchange for the acceptance of a small risk that the index file is corrupted which will lead to a failure if the corrupted entries are read and a message saying the index needs to be rebuilt. This can be safely switched on for ZFS on linux as the filesystem takes care of file checksums.

### MaxAutoMergeIndexLevel
`MaxAutoMergeIndexLevel` allows the maximum index file level that will be automatically merged to be specified. By default all levels will be merged. Depending on the specification of the host running Event Store, at some point it index merges will use a very large amount of disk IO. For example, 2 level 7 files being merged will result in at least 3072MB of reads (2 * 1536MB) and 3072MB writes. Setting this to level 7 will be automatically merged, but to merge the level 7 files together a manual merge needs to be triggered. This allows better control over when these larger merges happen and which nodes they happen on (due to the replication process, all nodes tend to merge at about the same time).

### OptimizeIndexMerge
`OptimizeIndexMerge` allows faster merging of indexes when a chunk has been scavenged. This option has no effect on unscavenged chunks. When a chunk has been scavenged and this option is set to true, a bloom filter will be used before reading the chunk to see if the value exists before reading the chunk to make sure that it still exists.

<!-- # todo: the 64 bit index bits should probably come under this indexing doc -->