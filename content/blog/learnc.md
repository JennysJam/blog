+++
title = "What I wished I knew when learning C"
+++

This article is a response/take on both [Daniel Bachler's article on learning F#](https://danielbachler.de/2020/12/23/what-i-wish-i-knew-when-learning-fsharp.html) and [Hillel Wayne's on the hard part of learning a new language](https://www.hillelwayne.com/post/learning-a-language/). I thought it would be educational and a fun excersize to read this article in response to C (and not C++, although I'll make notes about or mention it).

# How do I install the tools to write the language?

Unfortunately, C predates much of the model that Go or Rust or even Dotnet use, where there's a primary tool that exists standalone and handles not only compilation, package management and running; these features are quite fractured and depend on OS and what tooling your focusing on.

## The toolchain proper

The primary toolchains in use are GCC, Clang (using the LLVM infrastructure) and Windows MSVC. Several other toolchains and compilers exist, either as minimal focused projects like TCC, embedded toolchains for specific platforms like Intel's.

### On Linux

On most Linux's, the compilers are normally avaliable through the package manager (`apt`, `pacman`, etc), although which version depends on the version of your OS, and sometimes there are several versions of the library avaliable. 

Additionally, both GCC and Clang provides source code and premade binaries (TODO: links)