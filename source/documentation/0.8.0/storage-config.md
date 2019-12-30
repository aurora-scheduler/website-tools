# Storage Configuration And Maintenance

- [Overview](#overview)
- [Scheduler storage configuration flags](#scheduler-storage-configuration-flags)
  - [Mesos replicated log configuration flags](#mesos-replicated-log-configuration-flags)
    - [-native_log_quorum_size](#-native_log_quorum_size)
    - [-native_log_file_path](#-native_log_file_path)
    - [-native_log_zk_group_path](#-native_log_zk_group_path)
  - [Backup configuration flags](#backup-configuration-flags)
    - [-backup_interval](#-backup_interval)
    - [-backup_dir](#-backup_dir)
    - [-max_saved_backups](#-max_saved_backups)
- [Recovering from a scheduler backup](#recovering-from-a-scheduler-backup)
  - [Summary](#summary)
  - [Preparation](#preparation)
  - [Cleanup and re-initialize Mesos replicated log](#cleanup-and-re-initialize-mesos-replicated-log)
  - [Restore from backup](#restore-from-backup)
  - [Cleanup](#cleanup)

## Overview

This document summarizes Aurora storage configuration and maintenance details and is
intended for use by anyone deploying and/or maintaining Aurora.

For a high level overview of the Aurora storage architecture refer to [this document](/documentation/0.8.0/storage/).

## Scheduler storage configuration flags

Below is a summary of scheduler storage configuration flags that either don't have default values
or require attention before deploying in a production environment.

### Mesos replicated log configuration flags

#### -native_log_quorum_size
Defines the Mesos replicated log quorum size. See
[the replicated log configuration document](/documentation/0.8.0/deploying-aurora-scheduler/#replicated-log-configuration)
on how to choose the right value.

#### -native_log_file_path
Location of the Mesos replicated log files. Consider allocating a dedicated disk (preferably SSD)
for Mesos replicated log files to ensure optimal storage performance.

#### -native_log_zk_group_path
ZooKeeper path used for Mesos replicated log quorum discovery.

See [code](https://github.com/apache/aurora/blob/#{git_tag}/src/main/java/org/apache/aurora/scheduler/log/mesos/MesosLogStreamModule.java)) for
other available Mesos replicated log configuration options and default values.

### Backup configuration flags

Configuration options for the Aurora scheduler backup manager.

#### -backup_interval
The interval on which the scheduler writes local storage backups.  The default is every hour.

#### -backup_dir
Directory to write backups to.

#### -max_saved_backups
Maximum number of backups to retain before deleting the oldest backup(s).

## Recovering from a scheduler backup

- [Overview](#overview)
- [Preparation](#preparation)
- [Assess Mesos replicated log damage](#assess-mesos-replicated-log-damage)
- [Restore from backup](#restore-from-backup)
- [Cleanup](#cleanup)

**Be sure to read the entire page before attempting to restore from a backup, as it may have
unintended consequences.**

### Summary

The restoration procedure replaces the existing (possibly corrupted) Mesos replicated log with an
earlier, backed up, version and requires all schedulers to be taken down temporarily while
restoring. Once completed, the scheduler state resets to what it was when the backup was created.
This means any jobs/tasks created or updated after the backup are unknown to the scheduler and will
be killed shortly after the cluster restarts. All other tasks continue operating as normal.

Usually, it is a bad idea to restore a backup that is not extremely recent (i.e. older than a few
hours). This is because the scheduler will expect the cluster to look exactly as the backup does,
so any tasks that have been rescheduled since the backup was taken will be killed.

### Preparation

Follow these steps to prepare the cluster for restoring from a backup:

* Stop all scheduler instances

* Consider blocking external traffic on a port defined in `-http_port` for all schedulers to
prevent users from interacting with the scheduler during the restoration process. This will help
troubleshooting by reducing the scheduler log noise and prevent users from making changes that will
be erased after the backup snapshot is restored

* Next steps are required to put scheduler into a partially disabled state where it would still be
able to accept storage recovery requests but unable to schedule or change task states. This may be
accomplished by updating the following scheduler configuration options:
  * Set `-mesos_master_address` to a non-existent zk address. This will prevent scheduler from
    registering with Mesos. E.g.: `-mesos_master_address=zk://localhost:2181`
  * `-max_registration_delay` - set to sufficiently long interval to prevent registration timeout
    and as a result scheduler suicide. E.g: `-max_registration_delay=360min`
  * Make sure `-gc_executor_path` option is not set to prevent accidental task GC. This is
    important as scheduler will attempt to reconcile the cluster state and will kill all tasks when
    restarted with an empty Mesos replicated log.

* Restart all schedulers

### Cleanup and re-initialize Mesos replicated log

Get rid of the corrupted files and re-initialize Mesos replicate log:

* Stop schedulers
* Delete all files under `-native_log_file_path` on all schedulers
* Initialize Mesos replica's log file: `mesos-log initialize <-native_log_file_path>`
* Restart schedulers

### Restore from backup

At this point the scheduler is ready to rehydrate from the backup:

* Identify the leading scheduler by:
  * running `aurora_admin get_scheduler <cluster>` - if scheduler is responsive
  * examining scheduler logs
  * or examining Zookeeper registration under the path defined by `-zk_endpoints`
    and `-serverset_path`

* Locate the desired backup file, copy it to the leading scheduler and stage recovery by running
the following command on a leader
`aurora_admin scheduler_stage_recovery <cluster> scheduler-backup-<yyyy-MM-dd-HH-mm>`

* At this point, the recovery snapshot is staged and available for manual verification/modification
via `aurora_admin scheduler_print_recovery_tasks` and `scheduler_delete_recovery_tasks` commands.
See `aurora_admin help <command>` for usage details.

* Commit recovery. This instructs the scheduler to overwrite the existing Mesosreplicated log with
the provided backup snapshot and initiate a mandatory failover
`aurora_admin scheduler_commit_recovery <cluster>`

### Cleanup
Undo any modification done during [Preparation](#preparation) sequence.

