+++
title = "CMake: The Good Parts"
date = 2024-11-21
description = "CMake: Pros, Cons and Weird Parts"
slug="cmake-pros"
+++

# CMake

CMake is one of the most popular build systems in the C/++ ecosystem according to a few recent developer survey ( [isocpp](https://isocpp.org/blog/2024/04/results-summary-2024-annual-cpp-developer-survey-lite), [CLion](https://www.jetbrains.com/lp/devecosystem-2023/cpp/#cpp_projectmodels_two_years) ).

At the same time, it's kind of an infamous one -- anytime I see it mentioned, I see a lot of negative feelings and opinions toward the language. I think it's important to acknowledge peoples feelings and where they are coming from and their frustrations; I have certainly ranted to my friends about CMake when I've been stumped by it before. 

But I also find the build system useful, so inspired by the [Lua: The Good, the Bad, The Ugly](https://notebook.kulchenko.com/programming/lua-good-different-bad-and-ugly-parts) I'm going to try to articulate my thoughts

## The Good

* CMake is _extremely_ backwards compatible, and via the [policy]() mechanism forward compatible as well.
* CMake has a ton of flexibility and builtin integration with a surprising number of systems and tools.
* Custom targets and actions allow for implementing more non-standard actions.
* Being a fully developed (if idiosyncratic) language is useful for code re-use: if creating a new target in you're project involves a very large or non-standard amount of boilerplate, you can use macros or functions for setup. LLVM does this.
* In general, I find that once you've done all of the complicated setup and the pages you make to the build system are adding new files or tweaking flags, I find that CMake feels much more concise than Makefiles (the only other build system I have major experience with). At the same time, I think CMake allows for much more complicated logic involving flags or target than Make does.
* CMake supports really solid utility for consuming targets and libraries, whether that is through a source package manager like [VCM]() or through use of git submodules, or if you have libraries installed on you're system with appropriate CMake scripts.
* CMake has a very rich set of release management tools under the `cpack` tool, allowing export to a very large number of formats.
* CMake also has a test runner utility system under `ctest`. I haven't used it, but it looks like it could be quite useful.


## The Weird

* CMake shares some influence (but no code pedigree) with TCL [based on some comments by developers](https://aosabook.org/en/v1/cmake.html). This mostly manifests in the stringly typed nature of the value system: values can be strings, numbers, and/or list of strings depending on the context.
* The other influence is that every builtin CMake command (and possibly user functions) receives it's arguments as strings after variable expansion; this effectively means every command is it's own DSL, which is why some special syntactic forms, like `LINKER:` in `target_link_options()`. This is also why variable evaluation works differently in `if()` conditions -- the attempted expansion of strings without any leading `$` sigil happens after the arguments are passed to the function.
* CMake functions and macros cannot return values directly, so they must pass variables names in the outer scope that the function will set and expand. 
* At the same time, [generator expressions]() entirely live inside their own expression oriented scope, and sometimes require you to write out stuff that doesn't feel natural for them to evaluate correctly.
* Generator Expressions also are only allowed (or expanded) inside some particular commands.

## The Bad

* CMake is very, very complicated and takes a lot of time to learn.
* It feels like CMake has a lot of necessary boiler plate and defensive code required, especially if you are starting a new project.
* In my experience, it's very easy to accidentally make code that works in one use case, but not another, and errors are often implicit and unclear in those cases.
* A lot of the online help about CMake is not super good, and often times code snippets I find on Stackoverflow are not useful or are counterproductive. I absolutely do hold this against those development authors! Programming is hard, and I'm sure most have found useful.