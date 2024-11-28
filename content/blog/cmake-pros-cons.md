+++
title = "CMake: The Good, The Bad, The Weird"
date = 2024-11-21
description = "CMake: Pros, Cons and Weird Parts"
slug="cmake-pros"
+++

# CMake: the Good, the Bad, the Weird

CMake is one of the most popular build systems in the C/++ ecosystem according to a few recent developer surveys ([IsoCPP survey](https://isocpp.org/blog/2024/04/results-summary-2024-annual-cpp-developer-survey-lite), [CLion Survey](https://www.jetbrains.com/lp/devecosystem-2023/cpp/#cpp_projectmodels_two_years)).

At the same time, it's kind of an infamous one -- anytime I see it mentioned, I see a lot of negative feelings and opinions toward the language. I think it's important to acknowledge peoples feelings and where they are coming from and their frustrations -- I have certainly ranted to my friends about CMake when I've been stumped by it before!

But I also find the build system useful, so inspired by the [Lua: The Good, the Bad, The Ugly](https://notebook.kulchenko.com/programming/lua-good-different-bad-and-ugly-parts) I'm going to try to articulate my thoughts on what's good about it, what's bad about it, and what's kind of just a weird quirk about it.

I also strongly recommend for any developers who are learning or struggling with CMake to purchase [Professional CMake](https://crascit.com/professional-cmake/) -- I have found it very helpful in explaining things where most other resources haven't, and it is consistently updated with new major versions of CMake.

## The Good

* CMake is _extremely_ backwards compatible, and via the [policy](https://crascit.com/professional-cmake/) mechanism forward compatible as well.
* CMake has a ton of flexibility and builtin integration with a surprising number of systems and tools like Doxygen, Python, Package Config.
* Custom targets and actions allow for implementing more non-standard actions.
* Relatedly, the `cmake` binary [itself](https://cmake.org/cmake/help/latest/manual/cmake.1.html#run-a-command-line-tool) acts as a sort of very light [busybox](https://busybox.net/) tool, allowing custom targets to not rely on system-specific names of tools.
* Being a fully developed (if idiosyncratic) language is useful for code re-use: if creating a new target in you're project involves a very large or non-standard amount of boilerplate, you can use macros or functions for setup. LLVM does this for in-tree passes and it seems fairly pleasant to use.
* In general, I find that once you've done all of the complicated setup and the pages you make to the build system are adding new files or tweaking flags, I find that CMake feels much more concise than Makefiles (the only other build system I have major experience with). At the same time, I think CMake allows for much more complicated logic involving flags or target than Make does and it tends to be a lot more readable when done this way.
* CMake supports really solid utility for consuming targets and libraries, whether that is through a source package manager like VCPKG, vendored-in subprojects, git submodules, or if you have libraries installed on you're system with appropriate CMake scripts.
* CMake can fetch dependencies at configure time using [`FetchContent`](https://cmake.org/cmake/help/latest/guide/using-dependencies/index.html#downloading-and-building-from-source-with-fetchcontent).
* This gives CMake a huge network effect -- most C++ (but not as many C) libraries I'm aware of have CMake files to allow them to be integrated like this. 
* CMake has a very rich set of release management tools under the `cpack` tool, allowing export to a [large set of formats](https://cmake.org/cmake/help/latest/manual/cpack-generators.7.html#manual:cpack-generators(7)).
* CMake also has a test runner utility system under `ctest`. I haven't used it, but it looks like it could be quite useful, and an associated dashboard under `cdash`.
* The CMake binary can actually be used to launch a build itself, through `cmake --build <build> -j <num-threads>`. This is very convenient and how I almost always invoke CMake builds.
* On Windows, CMake will attempt to locate a MSVC install and use it even if you are not inside the Visual Studio developer environment, which smooths out testing out builds on Windows a lot in my experience.
* `compile-commands.json` can be output by CMake, which gives IDEs and other tooling introspection into how a build works.

## The Weird

* In general, CMake is a "have the newest version possible installed on your system" type tool, I almost always install it through official release instead of a package manager.
* CMake shares some influence (but no code pedigree) with TCL [based on some comments by developers](https://aosabook.org/en/v1/cmake.html). This mostly manifests in the stringly typed nature of the value system: values can be strings, numbers, and/or list of strings depending on the context.
* The other influence is that every builtin CMake command (and possibly user functions) receives it's arguments as strings after variable expansion; this effectively means every command is it's own DSL, which is why some special syntactic forms, like `LINKER:` in `target_link_options()`. This is also why variable evaluation works differently in `if()` conditions -- the attempted expansion of strings without any leading `$` sigil happens after the arguments are passed to the function.
* CMake functions and macros cannot return values directly, so they must pass variables names in the outer scope that the function will set and expand. 
* At the same time, [generator expressions](https://cmake.org/cmake/help/latest/manual/cmake-generator-expressions.7.html) entirely live inside their own expression oriented scope, and sometimes require you to write out stuff that doesn't feel natural for them to evaluate to what you want them to.
* Generator Expressions also are only allowed (or expanded) inside some particular commands.
* CMake has two types of variable storage: "normal" variables that exist while configuration is running, and "cache" variables that are stored in a flat file database called a cache -- declaration of cache variables do not change their value if the flat file exists and that variable is defined, but syntactically they are treated almost equivalently.
* CMake policies and some actions are not evaluated until the end of the file when the built up dependency tree is evaluated (unless you're using a policy scope? maybe?) so sometimes code seems to be executing out of order with little warning.
* I believe builds for the Apple ecosystem were added much later, and thus often times have custom behavior you have to be aware of when targeting them. I wouldn't be surprised if most build scripts I wrote didn't work with XCode!
* Any change to configure code requires you to rerun the configure, and potentially delete the build directory and restart -- often meaning tweaking small details can require full builds.

## The Bad

* CMake is very, very complicated and takes a lot of time to learn -- I spent almost a week just reading technical documentation trying to get a sense of how a lot of systems works, and I still have tons of blindspots.
* It feels like CMake has a lot of necessary boiler plate and defensive code required, especially if you are starting a new project. I personally find this to be enough friction that experimental or prototype projects normally start out with Makefiles or a shell script instead.
* In my experience, it's very easy to accidentally write a build script that works in one use case, but not in a very similar case. Often this is because of how the test of the underlying build system output is run and thus not detectable at configure time, but still quite difficult.
* A lot of the online help about CMake is not super good, and often times code snippets I find on Stackoverflow are not useful or are counterproductive. I absolutely do not hold this against those development authors! Programming is hard, and I'm sure most have found the tips they offered useful, it's just that they often don't work correctly in all use cases.
* There are lots of just little inconsistencies, where two features that can interact don't work like they seem to be, or sometimes a specific command cannot handle a relative path. These all tend to be small things, but I'm sure every developers whose lost an hour on them absolutely remembers them.
* Generator Expressions feel like a later addition to the language that do not mesh very well with it, and writing CMake code that handles being built in a multi-config build requires a lot of changes to how code obviously feels like it should be done with in.
* In general, writing a CMake project for a library involves an even higher level of ensuring quality and that things work. I think if you are developing a library with a large userbase it's worth the effort, but it absolutely is an effort.
* The provenance of variables is sometimes hard to find: sometimes it's set inside global code in a way you can't change, or involves string editing the various build flag globals. This has bitten at least a few developers!
* CMake historically utilized tons of global various, but with the introduction of the 3.X versions of the tool, a more OOP-like system of creating targets with `add_executable()` et al and editing them with `target_()` commands.

## So, is CMake worth it?

It depends on you and your project!

I think Cmake has a ton of benefits, but a huge learning curve and plenty of issues. I think the friction the defensive boilerplate coding required for CMake is maybe not worth it for exploratory or prototype code, but at the same time I think the benefits for very large codebases are worth it. 

At the same time, if you find yourself doing just fine with Makefiles, or Meson, or SCons, or Bazel, or xmake -- keep doing what you're doing! Build systems are to help build a project, and if you're use case is being met then I don't think you need to switch over to a new build system for hazy network effects.

It's totally possible that in the future, CMake becomes a legacy thing that has a similar reputation to autotools, but for now it is if anything growing in use and adoption.

I also personally think build tools need more love: it's easy to ignore them because they are cost centers for developing code, but they are a vital part of the development, release and management lifecycle.
