## Update the MySQL HA Cluster

This role is designed to update the MySQL HA cluster to a new minor version (for example, 15.1->15.2, and etc).

By default, only MySQL packages defined in the MySQL_packages variable are updated (vars/Debian.yml or vars/RedHat.yml). In addition, you can update Orchestrator or the entire system. 

#### Usage

Update MySQL:

`ansible-playbook update_pgcluster.yml`

Update Orchestrator:

`ansible-playbook update_pgcluster.yml -e target=Orchestrator`

Update all system packages:

`ansible-playbook update_pgcluster.yml -e target=system`


#### Variables

- `target` 
  - Defines the target for the update.
  - Available values: 'postgres', 'Orchestrator', 'system'
  - Default value: postgres
- `max_replication_lag_bytes`
  - Determines the size of the replication lag above which the update will not be performed.
  - If the lag is high, you will be prompted to try again later.
  - Default value: 10485760 (10 MiB)
- `max_transaction_sec`
  - Determines the maximum transaction time, in the presence of which the update will not be performed.
  - If long-running transactions are present, you will be prompted to try again later. 
  - Default value: 15 (seconds)
- `update_extensions`
  - If 'true', an attempt will be made to automatically update all extensions for all databases.
  - Specify 'false', to avoid updating extensions.
  - Default value: true
---

## Plan:

Note: About the expected downtime of the database during the update:

When using load balancing for read-only traffic (the "Type A" and "Type C" schemes), zero downtime is expected (for read traffic), provided there is more than one replica in the cluster. For write traffic (to the Primary), the expected downtime is ~5-10 seconds.

#### 1. PRE-UPDATE: Perform Pre-Checks
- Test MySQL DB Access
- Make sure that physical replication is active
  - Stop, if there are no active replicas
- Make sure there is no high replication lag
  - Note: no more than `max_replication_lag_bytes`
  - Stop, if replication lag is high
- Make sure there are no long-running transactions
  - no more than `max_transaction_sec`
  - Stop, if long-running transactions detected
#### 2. UPDATE: Secondary (one by one)
- Stop read-only traffic
  - Enable `noloadbalance`, `nosync`, `nofailover` parameters in the Orchestrator.yml
  - Reload Orchestrator service
  - Make sure replica endpoint is unavailable
  - Wait for active transactions to complete
- Stop Services
  - Execute CHECKPOINT before stopping MySQL
  - Stop Orchestrator service on the Cluster Replica
- Update MySQL
  - if `target` variable is not defined or `target=postgres`
  - Install the latest version of MySQL packages
- Update Orchestrator
  - if `target=Orchestrator` (or `system`)
  - Install the latest version of Orchestrator package
- Update all system packages (includes MySQL and Orchestrator)
  - if `target=system`
  - Update all system packages
- Start Services
  - Start Orchestrator service
  - Wait for Orchestrator port to become open on the host
  - Check that the Orchestrator is healthy
  - Check MySQL is started and accepting connections
- Start read-only traffic
  - Disable `noloadbalance`, `nosync`, `nofailover` parameters in the Orchestrator.yml
  - Reload Orchestrator service
  - Make sure replica endpoint is available
- Perform the same steps for the next replica server.
#### 3. UPDATE: Primary
- Switchover Orchestrator leader role
  - Perform switchover of the leader for the Orchestrator cluster
  -  Make sure that the Orchestrator is healthy and is a replica
     - Notes:
       - At this stage, the leader becomes a replica
       - the database downtime is ~5 seconds (write traffic)
- Stop read-only traffic
  - Enable `noloadbalance`, `nosync`, `nofailover` parameters in the Orchestrator.yml
  - Reload Orchestrator service
  - Make sure replica endpoint is unavailable
  - Wait for active transactions to complete
- Stop Services
  - Execute CHECKPOINT before stopping MySQL
  - Stop Orchestrator service on the old Cluster Leader
- Update MySQL
  - if `target` variable is not defined or `target=postgres`
  - Install the latest version of MySQL packages
- Update Orchestrator
  - if `target=Orchestrator` (or `system`)
  - Install the latest version of Orchestrator package
- Update all system packages (includes MySQL and Orchestrator)
  - if `target=system`
  - Update all system packages
- Start Services
  - Start Orchestrator service
  - Wait for Orchestrator port to become open on the host
  - Check that the Orchestrator is healthy
  - Check MySQL is started and accepting connections
- Start read-only traffic
  - Disable `noloadbalance`, `nosync`, `nofailover` parameters in the Orchestrator.yml
  - Reload Orchestrator service
  - Make sure replica endpoint is available
#### 4. POST-UPDATE: Update extensions
- Update extensions
  - Get the current Orchestrator Cluster Leader Node
  - Get a list of databases
  - Update extensions in each database
    - Get a list of old MySQL extensions
    - Update old MySQL extensions (if an update is required)
- Check the Orchestrator cluster state
- Check the current MySQL version
- List the Orchestrator cluster members
- Update completed.
