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
