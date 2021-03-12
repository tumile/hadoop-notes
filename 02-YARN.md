## MapReduce

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

## YARN

- Split up resource amangement and job scheduling/monitoring into separate daemons
  - application is either a single job or a DAG of jobs
  - global ResourceManager
    - arbitrate resources among all applications in the system
  - per-node NodeManager
    - framework agent monitors containers, resource usage and reports to ResourceManager
  - per-application ApplicationMaster
    - framework specific library negotiates resources from ResourceManager and works with NodeManager(s) to execute and monitor tasks
- ResourceManager has
  - Scheduler
    - allocate resources to applications
    - no monitoring, tracking, restarting
    - perform scheduling function based on resource requirements of applications, based on abstract Container
    - has pluggable policies for partitioning the cluster resources among applications
  - ApplicationsManager
    - accept job-submissions, negotiate the first container for executing the ApplicationMaster
    - restart ApplicationMaster on failure
    - ApplicationMaster negotiates appropriate resource containers from Scheduler, tracks status and monitors progress
- More YARN features
  - supports resource reservation, ensure predictable execution of important jobs
  - supports Federation, transparently wire together multiple YARN (sub)clusters
