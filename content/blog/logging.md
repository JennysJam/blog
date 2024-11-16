+++
title = "Thoughts on Logging"
date = 2024-11-09
description = "Thoughts on what application logging systems should add"
slug="logging"
+++

# Thoughts on Logging
I read two blog posts recently that made me think about blogging: A footnote in Hillel Wayne's [Why Not Comment](https://buttondown.com/hillelwayne/archive/why-not-comments/) and another blog post about continious improvement of codebases.

I've been thinking about logging for a second, and so alos had the thought of thinking about it.

## Log levels may be too granular

A lot of logging systems (in addition to allow custom logging) usually have a pretty large number of builtin log levels. [Python has 5](https://en.wikipedia.org/wiki/Syslog#Severity_level), [Syslog has 8](https://en.wikipedia.org/wiki/Syslog#Severity_level), and I've seen others move up to over 10. 

I feel like its hard to intuit what level you should or should be using with so many levels -- is an error a `WARN` or an `ERROR` or `CRITICAL`? 

I feel this also makes filtering out particular levels more difficult: aside from `VERBOSE`, it's hard to feel out what level something should be at.

Here's my personal, totally off my head cutting down of levels and semantics

* `ERROR` or `FAIL`: a core or important subsystem or operation failed, system likely should panic or shutdown.
* `WARN`: a subsystem failed or operation failed but the system can keep working, however this suggests code should be reviewed.
* `INFO`: the default/something is going on message. If something like a config file is not found but that's handleable or expected it should go here.
* `DEBUG`: verbose logging information for when something is going on.

If individual topics can have different logging levels, then `DEBUG` would be a good candidate for enabling for particular files or subsystems.

If you're building a very large or complete system, maybe an extra `CRITICAL` level means to suggest the entire system (like an OS?) is failing in a critical method.

## `TODO` or `DEV` Level

In my experience, a lot of developers writing logs are continioug their use as print debugging, to show information about the internals of the system. I think this is good! I do it too.

But it also means that when working with more complete or deployed code, a lot of the logs that show up are things that were incrementally added to debug a particular part of code that was problematic or having issues, but may not be as meaningful if you're triaging a whole system to get an idea of what's going on.

My half baked solution is to add a special log level: `TODO` or `DEV`. This level would either be the highet level or same level as the highest so it always appears.

Suggested practice would be that any developer working on a new feature would add all logging statements as `TODO` unless you're fairly confidant it should be shown up as a regular logging statmeent, and then during code review all `TODO` logs are either converted to their appropriate levels or removed. 

Maybe a custom a linting tool could be added to check for this?

On the other hand, rewriting code is more likely to add bugs, and those debugging log statements added while debuging code may be better to keep because the original developer is aware of them, and a rewrite may remove/get rid of implicit context the original person had?

## `SUCCESS` Level
The Python library [Loguru](https://github.com/Delgan/loguru) has an extra `SUCCESS` level. I thought this was a good idea to indicate some sizable sub-operation suceeded, especially in a response/request or command based system.

I also think it would make a good set of terms for .e.g. a web server:

```txt
INFO    recieved request for /some/random/endpoint
SUCCESS served response for /some/random/endpoint
```

which is nice as a pair for

```txt
INFO    recieved request for /buggy/endpoint
WARN   failed /buggy/endpoint -- invariant violation
```

## `OUT` as a logging level

This one is more hazy and maybe bad, but I also had the idea: if we're building up this complicated logging system, maybe regular `print()` printing to stdout could be an alias for `logger_subsystem::log(OUT, "whatever message")`, and then you could do the same level of feature/controls as other code does.

I'm really torn on this one, because normally logging is asumed to have some level of metadata or formatting, whereas standard output is generally more unstructured and can have essentially arbitrary code.