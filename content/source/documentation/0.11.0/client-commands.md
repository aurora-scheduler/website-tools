Aurora Client Commands
======================

- [Introduction](#introduction)
- [Cluster Configuration](#cluster-configuration)
- [Job Keys](#job-keys)
- [Modifying Aurora Client Commands](#modifying-aurora-client-commands)
- [Regular Jobs](#regular-jobs)
    - [Creating and Running a Job](#creating-and-running-a-job)
    - [Running a Command On a Running Job](#running-a-command-on-a-running-job)
    - [Killing a Job](#killing-a-job)
    - [Updating a Job](#updating-a-job)
        - [Coordinated job updates](#coordinated-job-updates)
    - [Renaming a Job](#renaming-a-job)
    - [Restarting Jobs](#restarting-jobs)
- [Cron Jobs](#cron-jobs)
- [Comparing Jobs](#comparing-jobs)
- [Viewing/Examining Jobs](#viewingexamining-jobs)
    - [Listing Jobs](#listing-jobs)
    - [Inspecting a Job](#inspecting-a-job)
    - [Versions](#versions)
    - [Checking Your Quota](#checking-your-quota)
    - [Finding a Job on Web UI](#finding-a-job-on-web-ui)
    - [Getting Job Status](#getting-job-status)
    - [Opening the Web UI](#opening-the-web-ui)
    - [SSHing to a Specific Task Machine](#sshing-to-a-specific-task-machine)
    - [Templating Command Arguments](#templating-command-arguments)

Introduction
------------

Once you have written an `.aurora` configuration file that describes
your Job and its parameters and functionality, you interact with Aurora
using Aurora Client commands. This document describes all of these commands
and how and when to use them. All Aurora Client commands start with
`aurora`, followed by the name of the specific command and its
arguments.

*Job keys* are a very common argument to Aurora commands, as well as the
gateway to useful information about a Job. Before using Aurora, you
should read the next section which describes them in detail. The section
after that briefly describes how you can modify the behavior of certain
Aurora Client commands, linking to a detailed document about how to do
that.

This is followed by the Regular Jobs section, which describes the basic
Client commands for creating, running, and manipulating Aurora Jobs.
After that are sections on Comparing Jobs and Viewing/Examining Jobs. In
other words, various commands for getting information and metadata about
Aurora Jobs.

Cluster Configuration
---------------------

The client must be able to find a configuration file that specifies available clusters. This file
declares shorthand names for clusters, which are in turn referenced by job configuration files
and client commands.

The client will load at most two configuration files, making both of their defined clusters
available. The first is intended to be a system-installed cluster, using the path specified in
the environment variable `AURORA_CONFIG_ROOT`, defaulting to `/etc/aurora/clusters.json` if the
environment variable is not set. The second is a user-installed file, located at
`~/.aurora/clusters.json`.

A cluster configuration is formatted as JSON.  The simplest cluster configuration is one that
communicates with a single (non-leader-elected) scheduler.  For example:

```javascript
[{
  "name": "example",
  "scheduler_uri": "localhost:55555",
}]
```

A configuration for a leader-elected scheduler would contain something like:

```javascript
[{
  "name": "example",
  "zk": "192.168.33.7",
  "scheduler_zk_path": "/aurora/scheduler"
}]
```

For more details on cluster configuration see the
[Client Cluster Configuration](/documentation/0.11.0/client-cluster-configuration/) documentation.

Job Keys
--------

A job key is a unique system-wide identifier for an Aurora-managed
Job, for example `cluster1/web-team/test/experiment204`. It is a 4-tuple
consisting of, in order, *cluster*, *role*, *environment*, and
*jobname*, separated by /s. Cluster is the name of an Aurora
cluster. Role is the Unix service account under which the Job
runs. Environment is a namespace component like `devel`, `test`,
`prod`, or `stagingN.` Jobname is the Job's name.

The combination of all four values uniquely specifies the Job. If any
one value is different from that of another job key, the two job keys
refer to different Jobs. For example, job key
`cluster1/tyg/prod/workhorse` is different from
`cluster1/tyg/prod/workcamel` is different from
`cluster2/tyg/prod/workhorse` is different from
`cluster2/foo/prod/workhorse` is different from
`cluster1/tyg/test/workhorse.`

Role names are user accounts existing on the slave machines. If you don't know what accounts
are available, contact your sysadmin.

Environment names are namespaces; you can count on `prod`, `devel` and `test` existing.

Modifying Aurora Client Commands
--------------------------------

For certain Aurora Client commands, you can define hook methods that run
either before or after an action that takes place during the command's
execution, as well as based on whether the action finished successfully or failed
during execution. Basically, a hook is code that lets you extend the
command's actions. The hook executes on the client side, specifically on
the machine executing Aurora commands.

Hooks can be associated with these Aurora Client commands.

  - `job create`
  - `job kill`
  - `job restart`

The process for writing and activating them is complex enough
that we explain it in a devoted document, [Hooks for Aurora Client API](/documentation/0.11.0/hooks/).

Regular Jobs
------------

This section covers Aurora commands related to running, killing,
renaming, updating, and restarting a basic Aurora Job.

### Creating and Running a Job

    aurora job create <job key> <configuration file>

Creates and then runs a Job with the specified job key based on a `.aurora` configuration file.
The configuration file may also contain and activate hook definitions.

### Running a Command On a Running Job

    aurora task run CLUSTER/ROLE/ENV/NAME[/INSTANCES] <cmd>

Runs a shell command on all machines currently hosting shards of a
single Job.

`run` supports the same command line wildcards used to populate a Job's
commands; i.e. anything in the `{{mesos.*}}` and `{{thermos.*}}`
namespaces.

### Killing a Job

    aurora job killall CLUSTER/ROLE/ENV/NAME

Kills all Tasks associated with the specified Job, blocking until all
are terminated. Defaults to killing all instances in the Job.

The `<configuration file>` argument for `kill` is optional. Use it only
if it contains hook definitions and activations that affect the
kill command.

### Updating a Job

There are several sub-commands to manage job updates:

    aurora update start <job key> <configuration file>
    aurora update info <job key>
    aurora update pause <job key>
    aurora update resume <job key>
    aurora update abort <job key>
    aurora update list <cluster>

When you `start` a job update, the command will return once it has sent the
instructions to the scheduler.  At that point, you may view detailed
progress for the update with the `info` subcommand, in addition to viewing
graphical progress in the web browser.  You may also get a full listing of
in-progress updates in a cluster with `list`.

Once an update has been started, you can `pause` to keep the update but halt
progress.  This can be useful for doing things like debug a  partially-updated
job to determine whether you would like to proceed.  You can `resume` to
proceed.

You may `abort` a job update regardless of the state it is in. This will
instruct the scheduler to completely abandon the job update and leave the job
in the current (possibly partially-updated) state.

#### Coordinated job updates

Some Aurora services may benefit from having more control over updates by explicitly
acknowledging ("heartbeating") job update progress. This may be helpful for mission-critical
service updates where explicit job health monitoring is vital during the entire job update
lifecycle. Such job updates would rely on an external service (or a custom client) periodically
pulsing an active coordinated job update via a
[pulseJobUpdate RPC](https://github.com/apache/aurora/blob/#{git_tag}/api/src/main/thrift/org/apache/aurora/gen/api.thrift)).

A coordinated update is defined by setting a positive
[pulse_interval_secs](/documentation/0.11.0/configuration-reference/#updateconfig-objects) value in job configuration
file. If no pulses are received within specified interval the update will be blocked. A blocked
update is unable to continue rolling forward (or rolling back) but retains its active status.
It may only be unblocked by a fresh `pulseJobUpdate` call.

NOTE: A coordinated update starts in `ROLL_FORWARD_AWAITING_PULSE` state and will not make any
progress until the first pulse arrives. However, a paused update (`ROLL_FORWARD_PAUSED` or
`ROLL_BACK_PAUSED`) is still considered active and upon resuming will immediately make progress
provided the pulse interval has not expired.

### Renaming a Job

Renaming is a tricky operation as downstream clients must be informed of
the new name. A conservative approach
to renaming suitable for production services is:

1.  Modify the Aurora configuration file to change the role,
    environment, and/or name as appropriate to the standardized naming
    scheme.
2.  Check that only these naming components have changed
    with `aurora diff`.

        aurora job diff CLUSTER/ROLE/ENV/NAME <job_configuration>

3.  Create the (identical) job at the new key. You may need to request a
    temporary quota increase.

        aurora job create CLUSTER/ROLE/ENV/NEW_NAME <job_configuration>

4.  Migrate all clients over to the new job key. Update all links and
    dashboards. Ensure that both job keys run identical versions of the
    code while in this state.
5.  After verifying that all clients have successfully moved over, kill
    the old job.

        aurora job killall CLUSTER/ROLE/ENV/NAME

6.  If you received a temporary quota increase, be sure to let the
    powers that be know you no longer need the additional capacity.

### Restarting Jobs

`restart` restarts all of a job key identified Job's shards:

    aurora job restart CLUSTER/ROLE/ENV/NAME[/INSTANCES]

Restarts are controlled on the client side, so aborting
the `job restart` command halts the restart operation.

**Note**: `job restart` only applies its command line arguments and does not
use or is affected by `update.config`. Restarting
does ***not*** involve a configuration change. To update the
configuration, use `update.config`.

The `--config` argument for restart is optional. Use it only
if it contains hook definitions and activations that affect the
`job restart` command.

Cron Jobs
---------

You can manage cron jobs using the `aurora cron` command.  Please see
[cron-jobs.md](/documentation/0.11.0/cron-jobs/) for more details.

You will see various commands and options relating to cron jobs in
`aurora -h` and similar. Ignore them, as they're not yet implemented.

Comparing Jobs
--------------

    aurora job diff CLUSTER/ROLE/ENV/NAME <job configuration>

Compares a job configuration against a running job. By default the diff
is determined using `diff`, though you may choose an alternate
 diff program by specifying the `DIFF_VIEWER` environment variable.

Viewing/Examining Jobs
----------------------

Above we discussed creating, killing, and updating Jobs. Here we discuss
how to view and examine Jobs.

### Listing Jobs

    aurora config list <job configuration>

Lists all Jobs registered with the Aurora scheduler in the named cluster for the named role.

### Inspecting a Job

    aurora job inspect CLUSTER/ROLE/ENV/NAME <job configuration>

`inspect` verifies that its specified job can be parsed from a
configuration file, and displays the parsed configuration.

### Checking Your Quota

    aurora quota get CLUSTER/ROLE

  Prints the production quota allocated to the role's value at the given
cluster. Only non-[dedicated](/documentation/0.11.0/deploying-aurora-scheduler/#dedicated-attribute)
[production](/documentation/0.11.0/configuration-reference/#job-objects) jobs consume quota.

### Finding a Job on Web UI

When you create a job, part of the output response contains a URL that goes
to the job's scheduler UI page. For example:

    vagrant@precise64:~$ aurora job create devcluster/www-data/prod/hello /vagrant/examples/jobs/hello_world.aurora
    INFO] Creating job hello
    INFO] Response from scheduler: OK (message: 1 new tasks pending for job www-data/prod/hello)
    INFO] Job url: http://precise64:8081/scheduler/www-data/prod/hello

You can go to the scheduler UI page for this job via `http://precise64:8081/scheduler/www-data/prod/hello`
You can go to the overall scheduler UI page by going to the part of that URL that ends at `scheduler`; `http://precise64:8081/scheduler`

Once you click through to a role page, you see Jobs arranged
separately by pending jobs, active jobs and finished jobs.
Jobs are arranged by role, typically a service account for
production jobs and user accounts for test or development jobs.

### Getting Job Status

    aurora job status <job_key>

Returns the status of recent tasks associated with the
`job_key` specified Job in its supplied cluster. Typically this includes
a mix of active tasks (running or assigned) and inactive tasks
(successful, failed, and lost.)

### Opening the Web UI

Use the Job's web UI scheduler URL or the `aurora status` command to find out on which
machines individual tasks are scheduled. You can open the web UI via the
`open` command line command if invoked from your machine:

    aurora job open [<cluster>[/<role>[/<env>/<job_name>]]]

If only the cluster is specified, it goes directly to that cluster's
scheduler main page. If the role is specified, it goes to the top-level
role page. If the full job key is specified, it goes directly to the job
page where you can inspect individual tasks.

### SSHing to a Specific Task Machine

    aurora task ssh <job_key> <shard number>

You can have the Aurora client ssh directly to the machine that has been
assigned a particular Job/shard number. This may be useful for quickly
diagnosing issues such as performance issues or abnormal behavior on a
particular machine.

### Templating Command Arguments

    aurora task run [-e] [-t THREADS] <job_key> -- <<command-line>>

Given a job specification, run the supplied command on all hosts and
return the output. You may use the standard Mustache templating rules:

- `{{thermos.ports[name]}}` substitutes the specific named port of the
  task assigned to this machine
- `{{mesos.instance}}` substitutes the shard id of the job's task
  assigned to this machine
- `{{thermos.task_id}}` substitutes the task id of the job's task
  assigned to this machine

For example, the following type of pattern can be a powerful diagnostic
tool:

    aurora task run -t5 cluster1/tyg/devel/seizure -- \
      'curl -s -m1 localhost:{{thermos.ports[http]}}/vars | grep uptime'

By default, the command runs in the Task's sandbox. The `-e` option can
run the command in the executor's sandbox. This is mostly useful for
Aurora administrators.

You can parallelize the runs by using the `-t` option.
