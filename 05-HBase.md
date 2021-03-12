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

Data in the memstore is sorted in the same manner as data in a HFile
When the memstore accumulates enough data, the entire sorted set is written to a new HFile in HDFS
Completing one large write task is efficient and takes advantage to HDFS’ strengths
HBase saves updates in a write-ahead-log (WAL) before writing the information to memstore
  -region server fails, information that was stored in that server’s memstore can be recovered from its WAL

client requests a change, that request is routed to a region server right away by default
Since the row key is sorted, it is easy to determine which region server manages which key. A change request is for a specific row. Each row key belongs to a specific region which is served by a region server
So based on the put or delete’s key, an HBase client can locate a proper region server. At first, it locates the address of the region server hosting the -ROOT- region from the ZooKeeper quorum.  From the root region server, the client finds out the location of the region server hosting the -META- region.  From the meta region server, then we finally locate the actual region server which serves the requested region.  This is a three-step process, so the region location is cached to avoid this expensive series of operations. If the cached location is invalid (for example, we get some unknown region exception), it’s time to re-locate the region and update the cache.