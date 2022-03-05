# Task Framework
I want you to design a task framework. This framework will trigger 1 billion tasks everyday. A task can be defined as a code executable. We want our task framework to support task runners in all modern programming languages. This task framework also needs to have a topological ordering, in which we can trigger subtasks and the sub tasks can trigger subtasks. Some tasks may be dependent on specific things but that is up for the user of our framework to implement. 

# Some constraints
A task of a given level in the task tree needs to be executing all possible tasks at the level similanously as to insure speed. we also need to be able to view results or metadata on these tasks and their execution. Keep in mind this is a framework that will be integrated and used in an existing architecture so we want to prioritze as little footprint as possible without sacrificing performance.
