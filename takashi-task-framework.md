# Task Framework

I want you to design a task framework. This framework will trigger 1 billion tasks everyday. A task can be defined as a code executable. We want our task framework to support task runners in all modern programming languages. This task framework also needs to have a topological ordering, in which we can trigger subtasks and the sub tasks can trigger subtasks. Some tasks may be dependent on specific things but that is up for the user of our framework to implement.

# Some constraints

A task of a given level in the task tree needs to be executing all possible tasks at the level simultaneously as to insure speed. we also need to be able to view results or metadata on these tasks and their execution. Keep in mind this is a framework that will be integrated and used in an existing architecture so we want to prioritize as little footprint as possible without sacrificing performance.

## Questions

1. What's a code executable? Like running a binary?

A: Could be a binary could be a python file. We also may want to support some yaml based system where we can trigger one executable after another in sequential order or in tandem.

2. Assuming that tasks can have subtasks and those subtasks can have subtasks infinitely, we'll have to fulfill our dependencies before we run the task at hand. Are there any other things to keep in mind other than subtasks, like requiring linked libraries for tasks? (Systemd allows you to require that some service is started or a library exists before running a binary, like C libraries require a libc before being run, since they're normally dynamically linked).

A: Yes dependency recovery is a real thing but keep in mind it falls on the user of the framework to provide code with valid libs.

3. How heavy are these tasks? How much hardware do we have? We need to run about 10k tasks per second, (1B / 10k seconds in a day).

A: This can depend we will probably need some sort of auto provisioning hardware to manage it.

4. Should I be providing some sort of interface that allows the users to ask for their dependencies up front like systemd does, or is that out of scope?

A: dependencies are pretty straightforward in most languages, lets say there is another team at bry-takashi-inc that is responsible for building a tool to pull dependencies from the file, but our system will still be responsible for installing those extracted dependencies

5. Do I need to care about analytics and logging?

A: YES of course we are big data organization

6. Can I assume that each task runs on one machine, or might tasks run on many?

A: Thats up to you and most likely depends on the task

7. What should I do if a task fails? Should I retry it in a bit, or can I bubble up an error to the user?

A: Retry in intervals each task will be configured with a number of retries

8. We can run tasks simultaneously, but is there any guarantee that they are idempotent? Is there any guarantee that they won't clash with the same data store? (e.g. what happens if task A wants to write X to a file, but task B wants to write Y to the same file. If task A gets the file first, and then task B overwrites the file, did task A succeed?) Should we do some sort of namespacing for each task to avoid this? If so, we'd sacrifice speed and memory, since we'd have to spin up an independent environment for each task.

A: Assume that the datastores these tasks are dependent are configured in such a way that they are incredibly scalable and will not have any issues with writes from multiple sources.

9. What sort of metadata should we be able to see? Like progress/success/failure, logging? What else would be necessary?

A: Success, Created, Pending, Failure thats up to you use some enum to represent the different task node states.

10. How much hardware will be available to us? Since we want to run a large number of jobs in parallel, we'll probably need many machines, hopefully stateless (although tasks might not be). If so, what happens if a machine dies or partially fails (hard disk corruption might happen at a large enough scale -- we expect 3 faulty drives per 100 ssds/year, so if we need 100 machines to run tasks for a year, there's going to be some non-zero percent of failure.

A: Assume we have a large datacenter with an api for provisioning resources and we are only responsible for writing the software for managing how much of these resources we use.

## Design

Assuming that we're trying to design a task framework that allows for
our users to run arbitrary things:

Let's start out with something like systemd:

First, we need some way to define a task: this part needs to be
sufficiently expressive for the end user.

It's important that our task runner allows for dependencies (like check
if lib-x is installed) and that it can check for the correct functionality
(lib-x must be able to do y). This part will be a lot like autotools,
where there are checks that occur before the task is run to make sure
the task runner is in a state where it can run the proper tasks.

One way to make this light without adding a lot of undue burden onto the
task running implementation is to allow shebang scripts for these
dependency checks.

Let's say we want to make sure that `pandoc` is on the `$PATH`, because
a task we want to describe requires pandoc.

// check-pandoc.sh

```sh
#!/usr/bin/env bash

which pandoc # returns a status code of 0 if found, 1 if not.
```

We can see that this command will return a `0` if `pandoc` is on the
`$PATH`, or a `1` if not.

Thus, we can compose our dependencies by having our task description run
bash scripts in the order they are placed in the config file, until they
all return 0 or one script returns a non-zero exit code. In case a
script returns a non-zero exit code, we can bubble up the error to the
user (clearly this task can't run).

Let's assume we'll use some key-value format, like an INI format.

```ini
[Checks]
check-pandoc="./check-pandoc"
```

Next, we want to make sure that we can pass environment variables, and
command line arguments to our script:

```ini
[Checks]
check-pandoc="./check-pandoc"

[Environment]
DOCKER_PORT="0.0.0.0"

[Args]
args="site.md -o site.html --template=_template.html"

[Command]
command="pandoc"
```

Finally, we need to find some way to express dependencies:

```ini
[Dependencies]
Requires=libpng.service # make sure that libpng is running.
After=network.target # make sure the network is running

[Service]
User=test
Group=test
WorkingDirectory=/tmp/pandoc
Restart=never
LimitMem=infinity # infinite memory
MaxTime=2s # if it doesn't return in 2s, kill this job

[Checks]
check-pandoc="./check-pandoc"

[Environment]
DOCKER_PORT="0.0.0.0"

[Args]
args="site.md -o site.html --template=_template.html"

[Command]
command="pandoc"
```

Where libpng.service might look like this:

```ini
[Dependencies]
Requires=libjpeg.service # make sure that libjpeg is running
```

So we can express dependencies of dependencies. We'll resolve
dependencies of dependencies by doing a topological sort, where we start
at the top of the dependency list and make sure they all successfully
run before going to the task at hand.

## Running Jobs

We'll also want a way to convey a task should be run periodically or
after something occurs: (e.g. an Account has too many inactive users
which need to be pruned for better performance on hot storage).

In the case of periodic jobs, we can support cron-like syntax in our INI
config files to allow for a task to fire every so often.

In the case of a condition being met, we'll allow the user to configure
how often they want to check for a particular condition and the action
they wish to do:

```ini
[RunIf]
Requires="./check-if-too-many-users.sh" # checks for user count of all
accounts, returns 0 if the task needs to run, 1 otherwise.

[Command]
command="./prune-users.sh"
```

Otherwise we can support a cron-like syntax:

```ini
[RunIf]
cron="* * * * *" # run every minute

[Command]
command="./heartbeat.sh" # check if the other server is still up.
```

We should also support a way to say how many times to restart a service
(if one is flaky, maybe we want to try 5 times). We also want to be able
to provide memory and disk limits, because otherwise tasks might (if
coded improperly) take down other tasks, which would be bad.

```ini
[Service]
Restart=5 # restart 5 times at most.
LimitMem=8G # allocate 8GB for this task.
LimitDisk=4G # allocate 4GB of memory for this task.
```

We can also "jail" each task, by using a FreeBSD-esque jail. Each task
will be granted its own process space, memory, and disk, and cannot
escape (in case malicious or fault code is executed, it won't be able to
cat /etc/passwd or other evil things).

## Logging

We need to make sure that logging is prioritized, since this is
complicated. If I write a task file and one of the dependencies fails to
install, this is a problem as well. We'll surface a command like
`task logs [$NAME]`
which will let the user query the task runner for the stdout of a task name.

## Running at Scale

Since we can assume that each task is idempotent (we'll jail each task
and assume they can run in any order, including in parallel), we can
allocate a cluster of machines to run tasks for us, and there can be a
centralized metadata service that users can query for the status of the
jobs they care about, along with audits of status changes, which they
can query by running `task audit [NAME]`.

We can assume that running 1B tasks per day is about 10k tasks per
second. We can try to keep a cluster of a thousand machines to run jobs,
all responding back to a centralized system that is replicated to store
metadata on tasks. The metadata server can generate ids for each task,
and also queries the tasks that need to run (cron jobs) and runs the
dependencies of all tasks provided, to know when to run a task if its
`RunIf` block wants it to run.

The centralized system could use a backing store of a time-series like
database, or it could use an append-only system, where it appends each
event that happens to a table, and a query will be indexed to make sure
it stays efficient (instead of table scanning).

We can create a status table that contains enums like so:
(this allows us to add more statuses in the future, just like errno in C).

```sql
create table status (
  id primary key auto increment,
  name varchar(255)
);
```

```json
{ "id": 1, "job_id": 3252, "status": 1, "timestamp": 17281 } # job 3252 completed at time 17281
{ "id": 2, "job_id": 3253, "status": 2, "timestamp": 17283 } # job 3253
is in progress
```

We can then query by `job_id` to see the status of jobs we care for, and
query with some SQL like syntax.

## MapReduce Jobs

We'll also run into the problem where some tasks must run on multiple
machines. If tasks can be run on multiple machines, we'll allow the user
to pass map-reduce like syntax (provide a map function), which the
centralized system will partition to open machines, and then finally a
reduce function which the centralized system will aggregate and then
finally write to a datastore (or a file).

We could allow for the user to specify that a task is a mapreduce job,
with the caveat that this requires tasks to be completely side effect
free, and our system must be able to rollback or otherwise kill the job
if it fails part way through (like a transaction).
