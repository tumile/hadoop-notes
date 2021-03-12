## Definition

- A framework to process big data in parallel on large cluster
  - MapReduce job splits the input dataset into chunks that are processed by map tasks
  - map results are sorted and compiled by reduce tasks
  - the framework takes care of scheduling, monitoring, re-executing tasks
- Typically compute nodes and storage nodes are the same, that is MapReduce and HDFS run on the same set of nodes
- MapReduce framework has
  - single master ResourceManager
  - one worker NodeManager per cluster-node
  - MRAppMaster per application
- Application defines input/output locations (HDFS files), map/reduce functions, and parameters -> job configuration
- Hadoop job client submits the job (jar/exe) to ResourceManager, which distributes to workers and monitors them
