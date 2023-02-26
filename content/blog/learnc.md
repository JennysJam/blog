+++
title = "What I wished I knew when learning C"
+++

This article is a response/take on both [Daniel Bachler's article on learning F#](https://danielbachler.de/2020/12/23/what-i-wish-i-knew-when-learning-fsharp.html) and [Hillel Wayne's on the hard part of learning a new language](https://www.hillelwayne.com/post/learning-a-language/). I thought it would be educational and a fun excersize to read this article in response to C (and not C++, although I'll make notes about or mention it).

# Why would I want to use C?

# Why would I not want to use C?

# Where can I run C?

# How do I install the tools to write the language?

Unfortunately, C predates much of the model that Go or Rust or even Dotnet use, where there's a primary tool that exists standalone and handles not only compilation, package management and running; these features are quite fractured and depend on OS and what tooling your focusing on.

## Downloading The toolchain proper

C toolchains come in a components; traditionally a compiler, an assembler, a linker. However it's common for the compiler to perform all together.

```shell
> gcc example.c secondary.c -o example
```

Performs compilation, code generatoin, and linking all in one step, and generally you pass special command line flags to indicate only to output object files, or to output assembly.

The primary toolchains in use are GCC, Clang (using the LLVM infrastructure) and Windows MSVC. Several other toolchains and compilers exist, either as minimal focused projects like TCC, embedded toolchains for specific platforms like Intel's.

### On Linux

On most Linux's, the compilers are normally avaliable through the package manager (`apt`, `pacman`, etc), although which version depends on the version of your OS, and sometimes there are several versions of the library avaliable. 

Additionally, both GCC and Clang provides source code and premade binaries (TODO: links) for several versions of the platform. For cross compilation, particular versions might matter, otherwise get the most recent version of the compiler that runs on your machine.

### On Windows

Generally anything developer facing in Windows land looks very different: Microsoft has for the longest time strongly linked together versions of both it's compiler, Microsoft Visual C, and it's IDE editor Visual Studio. Generally this will involve downloading the visual studio installer and choosing your particular version and which components you want.  This also will install the command line versions of those tools, although unlike on linux they are not added to the general environment -- instead you access them through a specified developer prompt that sets up environmental variables and adds the location of the toolchain.

Clang and GCC can both be built for windows, and may provide binaries, but both rely on the presnce of developer libraries and header files that are installed along with the MSVC compiler. 

# Debugging

# Packaging

# Code formatting

# Testing

# Common Gotchas