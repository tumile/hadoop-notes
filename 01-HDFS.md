# HDFS

## Assumptions and goals

- Hardware failure
  - something is always failing
  - quickly detects faults and automatically recovers
- Big data
  - theoretically unbounded size
  - high aggregate data bandwidth
  - volume is a built-in feature, not a challenge
- Streaming data access
  - designed for batch processing rather than real-time
  - emphasizes throughput over latency
  - write-once-read-many
  - append/truncate-only
  - not good for low-latency, random access

## Concepts

- Block
  - a unit of storage (128MB), files are splitted into block-size chunks
  - replicated to a number of physically separate machines
  - one replica per node
- Multiple DataNodes
  - actually store blocks of data
  - serve reads/writes from client or NameNode
    - tries to satisfy read from replica closest to the reader
  - reports to NameNode periodically with lists of their blocks
- NameNode: one per cluster
  - keeps metadata, filesystem namespace
    - stores locally 2 files: EditLog and FsImage
  - knows the DataNodes on which all the blocks for a given file are located
    - however does not store it persistently, because it's frequently changed
    - points client to the DataNode to read/write to
    - user data never flows through the NameNode
  - if the namenode is obliterated, everything would be lost since there is no way to reconstruct the files from the blocks on the DataNodes
- Keeping NameNode resilient
  - write NameNode state, synchronously and atomically, to multiple filesystems (NFS)
  - use Secondary NameNode/Checkpoint/Backup Node
  - use standby NameNode (high availability)
- Block Caching
  - blocks may be explicitly cached in the DataNode’s memory, in an off-heap block cache
  - can specify number of DataNodes that cache the block
  - job schedulers (Spark) can run tasks on cached DataNode for increased performance
- Federation
  - NameNode keeps a reference to every file and block in the filesystem in memory, which can limit scalability
  - under federation, multiple NameNodes can manage portions of the filesystem namespace
  - namespace volumes are independent
    - NameNodes do not communicate with one another
    - failure of one NameNode does not affect the availability of other namespaces
- High availability
  - because NameNode recovery time is long (minutes to hour)
  - use a hot standby NameNode, which can takes over quickly (tens of seconds)
    - two nodes use a shared storage to share edit logs: NFS filer or quorum journal manager (QJM)
  - QJM is dedicated HDFS highly available edit log (recommended)
    - has a group of journal nodes, each edit must write to a majority of nodes
    - not using Zookeeper, though HDFS HA does use Zookeeper for master election
- Failover and fencing
  - transition from acitve to standby is managed by the failover controller
    - a lightweight process monitoring failures and trigger failover
    - use Zookeeper to ensure only one NameNode is active
  - but it's possible that network lag causes false positive failover
    - HA implementation ensures the previously active NameNode is prevented from doing any damage, known as Fencing
    - QJM by design only allows one NameNode to write the edit log
- Safemode (read-only mode) on NameNode start-up
  - NameNode loads EditLog and FsImage
  - receives Heartbeat and Blockreport from DataNodes
  - replicates missing blocks (if any)

## Metadata

- Stored by NameNode, every changes is persistently logged in EditLog
  - e.g, creating new file, renaming, changing replication factor
- Filesystem namespace is stored in FsImage
  - e.g, mapping of blocks to files, file system properties
- NameNode keeps fs namespace and block map in memory
  - on start-up, reads EditLog and FsImage (safemode)
  - applies EditLog changes to FsImage, updates FsImage on disk and truncates EditLog; the process is called checkpoint
  - FsImage is essentially a snapshot
- DataNode stores blocks, but doesn't know about HDFS files
  - on start-up, it scans all blocks and send Blockreport to NameNode

## Robustness

- DataNode sends Heartbeat periodically
- Re-replication is tracked by NameNode and is needed when
  - DataNode unavailable, replica corrupted, replication factor increased
    - timeout to mark DataNode dead is conservatively long (10 mins) to avoid unnecessary replication
- Data integrity by checksums
  - comparing checksums on creating and retrieving files
- Files are divided into blocks on (if possible) different DataNodes
  - blocks are further replicated on different DataNodes
- When client writes a block to HDFS
  - NameNode returns a list of DataNodes for replication
  - client sends to the first DataNode, the first DataNode sends to the second, and so on; it's called a Write Pipeline

## Additional features

- Permission
  - much like POSIX model: read/write/execute, owner/group/world
- Authentication
- Balancer: tool to balance the cluster when data is unevenly distributed among DataNodes
- Secondary NameNode
  - primary NameNode performs checkpoint on start up, which can be slow
  - performs periodic checkpoints
  - stores check pointed image locally to be read by primary NameNode
- Checkpoint Node
  - same as Secondary NameNode, but check pointed image is uploaded to primary NameNode
  - there can be multiple Checkpoint Nodes
- Backup Node
  - same as Checkpoint Node
  - receives stream of edits from NameNode and keeps an in-memory copy of fs namespace, always in sync with primary NameNode
  - doesn't need to download EditLog and FsImage
  - more efficient as it only needs to save the namespace into the local FsImage file and reset edits
  - only 1 Backup Node allowed, no Checkpoint Node allowed when using Backup Node
  - allows running NameNode with no persistent storage, delegating all namespace persistance to Backup node
  - see more: https://issues.apache.org/jira/browse/HADOOP-4539

## API
- Hadoop has an abstract notion of filesystems, of which HDFS is just one implementation
- org.apache.hadoop.fs.FileSystem represents the client interface to a filesystem in Hadoop
  - there are implementations for Local, HDFS, WebHDFS, FTP, S3, Azure...

## Read flow
- Client here loosely refers to both the user application and the client library interacting with HDFS
- Client calls NameNode using RPC to get blocks locations, i.e DataNodes hosting them
- DataNodes are sorted by their proximity to the client
  - if the client itself is a DataNode (e.g. in a MapReduce task), the local block is read first
- Client reads directly from each DataNodes
  - if error occurs, try another DataNode
  - verifies checksums, reports corrupted blocks to NameNode
- NameNode merely has to service block location requests, which is very fast since it's in-memory

## Write flow
- Client calls NameNode to create a new file in the filesystem namespace, with no blocks
  - NameNode checks if the file exists and the client has adequate permissions
  - NameNode makes a record of the new file
- Client asks NameNode to allocate new blocks by picking a list of suitable DataNodes to store the replicas
- Client streams data to the first DataNode in the pipeline, which stores and forwards it to the second DataNode in the pipeline, and so on
  - if a DataNode fails, client and NameNode coordinate to create a new pipeline and ensure data is fully written and replicated
  - write is successful when a minimum number of replica is written, any missing will be asynchronously replicated

## Replica placement
How does NameNode choose DataNodes to replicate on?
https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HdfsDesign.html#Replica_Placement:_The_First_Baby_Steps
