## Definition

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
