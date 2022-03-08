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


## Core Functionality
1. Task Declaration Framework
1a. Yaml Definitions
1b. Security Layer
1c. Prepare task dependencies for execution
2. Autoscaling worker node system that provisions additional nodes/hardware for a given system.(Assume we are renting azure VMS and dont have to worry about the startup of these machines, we just need to handle the requesting of these machines.)
3. Task Dependency Manager
4. Task Metadata/Persistence Layer
### Task Declaration Framework
So first things first lets talk about what exactly I mean by task declaration framework.
 We want to be able to be able to execute some code, that will be written with our framework in mind.
 In a real world situation we would want to make a client framework available to allow these end users to interact with our framework in a programatic way possibly extending the framework to implement a job system. 
We can assume that one job may not be enough and that one job may depend on another job finishing.
 We also can assume we will only ever have depenedencies of a job and programatically it makes zero sense to have a cycle. 
So considering these things that the tasks need to have some form of topological order, it makes sense to represent the declaration of a task with some data type like yaml or json, where we can clearly add nested properties.
#### Yaml Keywords
So a given task definition needs to be represented in a way that is decipherable regardless of the language we are trying to execute.

Keep in mind its not our responsibility to have the tasks themselves function, thats up to the given user of the framework. 

So heres what i suggest. We have a variety of keywords that indicate a tasks responsibility. 


```yaml

task_name:
	task:
		schedule:300 # 0 to indicate run once else we specify the interval in which we want to run this task
		executable: https://github.com/liamg/furious/blob/master/main.go
		logger: syslog
		
		subtask_one:
			schedule: 0
			executable: https://github.com/liamg/gitjacker/blob/master/internal/pkg/gitjacker/retriever.go
		
		subtask_two:
			schedule: 0
			executable: link_to_some_python_file.py
			super_dependency:
				schedule: 0
				executable: link to ./kill_server.bash
```
So lets take a look at the given yaml and what information we can infer from it. 

we know the topological order of execution needs to be

```
super_dependency: 0
subtask_two: 1
subtask_one: 1
task: 2
```
We infer this from the yaml. Now we need to execute the tasks in this order, but you notice we have two tasks which are the same. For these tasks we can execute them concurrently.
So we first load the executables of the lowest indegree, then we execute them. But we have a problem. At the higher level we have a schedule of every 300 seconds. what if the execution of these 
tasks take longer than that 300 seconds? How can we avoid this? Well one solution to this type of problem would be to have some sort of autoprovisional component that vertically scales our current machine to allow for faster execution,
of tasks we predict will take longer. We can make estimates using the metadata of a previous run. If we are making estimates we need to estimate runtime across all machines. in other words we need an algorithm to come up with some O(N) 
time complexity analysis we can use to predict how long a given subtask will take. If we cannot execute a task in the given timeframe the user requests, even after scaling our machines as high as we can, or if a subtask is scheudled to run in an interval that is longer
than its parent, we need to throw some kind of error or change the status of a given task tree to failure.


#### Worker Autoscaling
So i talked a bit about some of the problems and ways we could address provisoning on a vertical level, but what if we have a massive amount of subtasks that have the same indegree? 
We need some sort of Distributed system to handle the horizontal scaling of these task nodes. Since we are only concerned with execution results and the most expensive part of this process is running the tasks(AKA its compute intensive rather than data intensive) 
we can keep all of the nodes in one region, maybe having backups of these workers in nearby locations (Austin, TX backup in nebraska). This way we dont have a lot of lag when communicating between the nodes asking if all of the nodes at a given level have reached 
a status of success. 


For a given tree we can have a counter.This counter stores the number of nodes of a given indegree in a task tree. Each time a node reaches success, we want to decrement this counter. we can have an expected completion time for each level of the tree based on task nodes of the same type. once the counter of a given indegree reaches zero, we get the ok to start executing the next level of nodes. We want to populate a queue of execution that each time we reach a new indegree, provisions an appropriate number of boxes to handle the tasks for that level. 


```python

# We can store the task statuses as numbers to save bytes and then have a enum or switch statement to handle returning the right status on the clientside of a task

def get_status(status: int):
	if status == 0:
		return "CREATED"
	elif status == 1:
		return "ENQUEUED"
	elif status == 2:
		return "STARTED"
	else:
		return "FAILURE"
```

A given task will be enqueued in execution in celery pulling nodes from redis, then we update some elasticsearch instance for each node.

```python
@dataclass
class TaskNode:
	status: int
	task_id: uuid
    context: json
	children: defaultdict(lambda: TaskNode) # task_id -> reference to the node in memory wherever we decide to store it.
```

