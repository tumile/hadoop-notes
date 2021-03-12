## Cluster setup

- Hadoop ecosystem has a common theme: master-slave architecture (except ZK)
- Master nodes
  - HDFS NameNode, YARN ResourceManager, and HBase Master
  - additional standby NameNode/ResourceManager for high availability setup
- Slave nodes
  - HDFS DataNodes, YARN NodeManagers, and HBase RegionServers
  - should be co-located or co-deployed for optimal data locality
- HBase requires ZK to manage the HBase cluster
- Separate master and slave nodes
  - task/application workloads on slaves should be isolated from masters
  - slaves are frequently and automatically replaced

### Examples

- Single node setup
  - all master and slave processes on the same machine
- 2-node setup
  - NameNode and ResourceManager on the master node, DataNode and NodeManager on the slave node
- 3 or more nodes setup
  - separate NameNode and ResourceManager nodes
  - all the other nodes are slaves
  - HA cluster adds additional standby NameNode/ResourceManager node
- Other services (Web App Proxy Server or MapReduce Job History) are either on dedicated or shared machine, depending on the load
