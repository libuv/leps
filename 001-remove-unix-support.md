
| Title  | Remove Unix support  |
|--------|----------------------|
| Author | @saghul              |
| Status | REJECTED             |
| Date   | 2014-11-26 21:05:50  |


## Overview

If we want libuv to succeed we need libuv to be Enterprise Ready (R). This means
Windows. Windows is *the* Operating System for The Enterprise. Linux and other Unix
systems are nice to play with but that's not where the money is.

When we start selling libuv licenses we'll need to do so to people able to pay for
them, like Windows users. Linux is obviously only used by hippies because it's free.
Moreover, people using open source have been associated with not so great corporal
hygiene, we don't wish to be associated with that.

Some may think we should also support OSX since Apple lovers tend to pay for licenses.
OSX has gone mainstream, but thorough analysis shows that most Macs are bought by
parents to their kids who can't afford them, so this is a no go.

This will reduce the code by about 11K lines, thus minimizing the maintenance burden
and reducing the supported operating systems to just The One.

## Course of action

In order to accomplish what's poposed here, the following steps will be followed:

* Remove all code dealing with Unix platforms
* Remove all build systems and use Visual Studio files
* Remove remaining abstractions
* Profit!

## Resons for rejection

This is obviously a joke.

