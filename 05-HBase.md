## Features

- Linear scalability
  - by adding RegionServers
  - if the cluster doubles the number of RegionServers, it doubles both in terms of storage and processing capacity
- Strongly consistent reads/writes
- Automatic and configurable sharding
  - tables are distributed via regions
  - regions are automatically split and re-distributed as data grows
- Automatic failover between RegionServers
- MapReduce support
  - using HBase as both input and output source
- Block Cache and Bloom Filters

## Data Model

- HBase is a map. Specifically it is a Sparse, Consistent, Distributed, Multidimensional, Sorted map.
  - Map: HBase maintains maps of Key to Value, called KeyValue or Cell
  - Sorted: cells are sorted by key
  - Multidimensional: key itself has structure
    - (rowkey, column family, column qualifier, timestamp) -> value
    - rowkey and value are just bytes (column family needs to be printable), so you can store anything that you can serialize into a byte[] into a cell.
    - imagine a map
    ```
    "rowkey1" : {
      "family1" : {
        "qualifier1" : {
          1 : "foo",
          2 : "fooo"
        },
        "qualifier2" : {
          15 : "bar",
        }
      },
      "family2" : {
        "qualifier3" : {
          ...
        }
      }
    },
    "rowkey2": {
      ...
    }
    ```
    - client asks for "rowkey:family:qualifier:timestamp" and get value; if no timestamp default to the latest
  - Sparse: a given row can have any number of columns in each column family, or none at all
  - Consitent: HBase makes two guarantees
    - all changes the with the same rowkey (see Multidimensional above) are atomic
    - a reader will always read the last written (and committed) values
- Rowkey is defined by the application. Combined key is prefixed by the rowkey, allowing the application to define the desired sort order
  - HBase ensures that all cells with the same rowkey are co-located on the same RegionServer, which allows ACID guarantees for updates with the same rowkey without complicated and slow two-phase-commit or paxos
- Column families are declared when a table is created. They define storage attributes such as compression, number of versions to maintain, TTL, and minimum number of versions, among others.
- Columns are arbitrary names (or labels) assigned by the application
- Timestamp is a long identifying (by default) the creation time of the of the cell
  - no data is ever overwritten or changed in place, instead every "update" creates a new version of the affected set of cells
- Concurrency
  - HBase ensures that all new versions created by single Put operation for a particular rowkey are either all seen by other clients or seen by none
  - Get or Scan will only return a combination of versions of cells for a row that existed together at some point. This ensures that no client will ever see a partially completed update or delete.
  - Achieve this by using a variation of Multiversion Concurrency Control
- Storage
  - Hbase stores cells. Cells are grouped by a rowkey into something that looks like a row. Cells are stored individually, so the storage is sparse
  - also means each cell carries its full coordinates (key). supported compression mitigates the repeated space to some extent
- Query
  - can only Scan a range of cells or Get a cells
- Updates are written to WAL, then to Memstore. When it reaches a certain size the Memstore is flushed to disk into HFile. Periodically HFile are compacted into fewer HFiles.
- Physically, all column family members are stored together on the filesystem, because tuning and storage specs are defined on column family level
- In short, Hbase tables are like tables in RDBMS, but cells are versioned, rows are sorted, column families are columns, and column qualifiers can be added on the fly

## HMaster and RegionServers

- Tables are automatically horizontally partitioned into regions
  - regions are automatically created as the table grows
  - contains a subset of table rows, denoted by table name, first row, and last row
- Hbase master bootstraps a virgin install, assigns regions to registered RegionServers, and recovers RegionServer failures
- Hbase uses Zookeeper to manage cluster state
  - by default manages its own ZK cluster, but can be configured to use an existing instance
  - stores location of the hbase:meta catalog table and the address of the current master
  - assignment of regions is meditated via ZK in case something crashes mid-assignment
  - client uses ZK to learn locations of Hbase servers

## hbase:meta

- Hbase keeps current state, locations of user-space regions on the cluster in hbase:meta table
- Entries are keyed by region name: table name, region start row, created time, MD5(all of those)
- Rowkeys are sorted, so finding the region containing a row is just looking up the largest entry whose key is >= rowkey
- Finding RegionServer
  - connects to ZK to find the location of hbase:meta, gets and caches hbase:meta
  - uses it to find hosting RegionServer of the key
  - interacts directly with the RegionServer
  - if there's a fault, i.e RegionServer has moved, checks the hbase:meta again
    - if hbase:meta has moved, checks ZK again

## ACID in Hbase

- Hbase has no mixed read/write transactions.
- Habse employs Multiversion Concurrency Control
  - each RegionServer maintains a strictly monotonically increasing transaction number
- Write flow
  - lock the row(s)
  - get the next highest transaction number - WriteNumber
  - apply changes to WAL
  - apply changes to Memstore, tagged with the WriteNumber
  - commit the transaction, i.e roll the ReadPoint forward to the WriteNumber
  - unlock the row(s)
- Read flow
  - open the scanner
  - get the transaction number of the last committed transaction - ReadPoint
  - filter all scanned KeyValues with memstore timestamp > the ReadPoint
  - close the scanner
- Reader acquires no locks
- Hbase commits transactions stricly serially
- A transaction's commit is delayed until all prior transactions committed
- HBase does not guarantee any consistency between regions
- During compactions, it's possible to see a partial row
  - Hbase keeps track of the earliest ReadPoint used by any open scanner and never collect any KeyValues with a memstore timestamp larger than that ReadPoint
  - supports ACID guarantees even with concurrent flushes

## Memstore and Hfile

- Data in the Memstore is sorted in the same manner as data in a HFile
  - internally it's a flavor of ConcurrentSkipListMap
- When the Memstore accumulates enough data, the entire sorted set is written to a new HFile in HDFS
  - completing one large write task is efficient and takes advantage to HDFS strengths
- HBase saves updates in a write-ahead-log (WAL) before writing the information to Memstore
  - recovery from server failures

## Hbase vs RDBMS

- HBase is a distributed, column-oriented data storage system (can be described as key-value as well)
- Provides random reads and writes on top of HDFS
  - designed from ground up with a focus on scale in every direction (billion rows, million columns)
  - easily horizontally partitioned and replicated
- RDBMS is fixed-schema, row-oriented, ACID, SQL query engine, strongly consistent, supports referential integrity, secondary indices, joins
- Only use Hbase for large amount of data, don't waste time for small DB

## Hbase vs MapReduce

- In MapReduce, files are opened and streamed through a map task and closed
- In Hbase, files are kept open to avoid the cost, so there are potential problems
  - runs out of file descriptors
  - runs out of DataNode threads
