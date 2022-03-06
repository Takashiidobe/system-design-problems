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
