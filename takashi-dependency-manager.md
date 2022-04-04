# Design a Dependency Manager

Design a dependency manager (a la homebrew, apt). This dependency
manager will support a list of commands to install (e.g. homebrew
install X) as well as support a config file for dependency maintainers
to build their software directly on the user's computer. (e.g. brew
install ruby runs a make file that compiles the ruby git repo on the
user's computer).

We'll also consider the backend, where we need to support a CDN like
infrastructure for cached assets that can be downloaded by users, and
also allow for users to install dependencies from a particular git
branch, or particular upstream (maybe they don't trust our centralized
one, and want to use their own private registry, like self-hosted npm or
maven).

## Questions

1. How do we guarantee to our users that their downloads haven't been
   tampered with before they make it to their computer?
2. How do we deal with users using different architectures and operating
   systems? (brew runs on x86_64, AArch64, and more architectures), and
   supports mac and linux for operating systems.
3. Should we allow direct binary downloads, or should we force all of
   our users to compile from source? Why is this important?
4. How do dependency packagers express that their package depends on
   certain things? How is this fraught with peril? What are some things
   to keep in mind while doing this?

## Features

Let's consider the three sides of the project:

1. Client
2. Maintainers
3. Backend (CDN)

### Client

The client should be able to do a few tasks:

1. Downloading a dependency
2. Query for downloaded dependencies
3. Remove a dependency

#### Downloading a dependency

To download a dependency, the client will connect to a CDN and download
either the source code (and compile the dependency on their machine) or
download a binary (if compiled) or the source code (if interpreted)
after checking for dependencies.

To download the required dependencies, the client should check the
dependencies required, and topological sort to resolve a dependency
chain required for the dependency.

It's important to make sure that the dependency works on the
architecture + OS + libc (target triplet) and all dependencies do as
well, before trying to download the target dependency.

The question about downloading source code or a binary is an important
design decision:

1. If we allow our users to download arbitrary source code and then
   compile code, package maintainers must audit the source code to make
   sure that it is correct.
2. If we allow package maintainers to post binaries, we need to audit
   binaries to make sure they're secure.

(We could allow the best of both worlds by having maintainers audit the
code of dependencies and then compile that and post that to the CDN).

However, we have some problems either way: since there are some critical
packages (those which are depended upon by lots of dependencies), a hack
into one of those will trickle down to affect all of our users, so
security is extremely important.

### Maintainers

Maintainers should be able to express which dependencies that their
dependencies require (e.g. dynamically linked libraries to run a
binary). They should be able to express a range of versions that these
dependencies can run on, and test this in CI to make sure that this
doesn't break some users. Tests should be required for this.

### Backend

Let's say that we allow our users to download either source code and
compile it themselves (maybe run a build script provided by the
maintainers) or a binary. In the case of source code, let's assume the
average code for a dependency is about 1MB, and the average binary is
5MB.

If we have 100k users who download one dependency, and each dependency
requires 9 new dependencies per download, our CDNs will need to take
~50MB per dependency. Assuming each person downloads 1 dependency
every day, each user will download about 50MB from our CDN per day.

At 50MB _ 100k users per day, we'll need to support a throughput of 50MB
in downloads a second. This will be about 1e6 _ 5MB, or about 5TB of
downloads a day. Assume it costs $0.01/GB to dl, so this should cost us
about $50 a day in network costs using a cloud provider.

Since most users are going to be downloading our assets instead of
updating them, it's fine to favor a CDN, since we'll be reading a lot
more than writing. In the case that CDNs are down, we may choose to have
a smaller backup CDN that can gradually increase in scale as it receives
traffic (in order not to waste much compute).
