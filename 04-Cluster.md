## Setup

- Master nodes
  - HDFS NameNode, YARN ResourceManager, and HBase Master
  - Standby NameNode for high availability setup
    - does checkpoint/backup node work
    - can replace active NameNode on failure
- Slave nodes
  - HDFS DataNodes, YARN NodeManagers, and HBase RegionServers
  - should be co-located or co-deployed for optimal data locality
- HBase requires ZooKeeper to manage the HBase cluster
- Separate master and slave nodes
  - task/application workloads on slaves should be isolated from masters
  - slaves are frequently and automatically replaced

- Single node setup
  - all master and slave processes on the same machine
- 2-node setup
  - NameNode and ResourceManager on the master node, DataNode and NodeManager on the slave node
- 3 or more nodes setup
  - separate NameNode and ResourceManager nodes
  - all the other nodes are slaves
  - HA cluster adds additional standby NameNode/ResourceManager node
- Other services (Web App Proxy Server or MapReduce Job History) are either on dedicated or shared machine, depending on the load

## HBase

https://cwiki.apache.org/confluence/display/ZOOKEEPER/HBaseUseCases

## Ambari
- Convention over configuration
- Setup https://cwiki.apache.org/confluence/display/AMBARI/Installation+Guide+for+Ambari+2.7.5
  - Install prerequisites (maven >3.3.9)
  - Fix node version in ambari-admin/pom.xml

wget -nv http://public-repo-1.hortonworks.com/ambari/centos7/2.x/updates/2.7.5.0/ambari.repo -O /etc/yum.repos.d/ambari.repo